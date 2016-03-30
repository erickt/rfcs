- Feature Name: `code_generation`
- Start Date: 2016-02-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a series of changes to the Rust compiler and Cargo in order
to better support code generators. It proposes the following high level
changes:

* Add source mapping support to the compiler that allows the compiler to
	bidirectionally associate tokens in an output rust file with one or more
	input template files.  This then will be used to report error messages in the
	original file.
* Add support to `rustc` for multiple source directories, and update Cargo
  to automatically add it's `$OUT_DIR` directory to this directory.
* Modify Cargo to enable libraries to register build scripts that remove
  the need for end users to maintain their own `build.rs` scripts.

# Motivation
[motivation]: #motivation

[Syntex](https://github.com/serde-rs/syntex) is a convenient tool that enables
libraries like [Serde](https://github.com/serde-rs/serde) to support Rust
Nightly-style syntax extensions in Stable Rust.  Syntex is a code generator,
where it expands syntax extensions from a template Rust file into a stable Rust
file.  This then can be compiled by the Stable Rust compiler.

Unfortunately there are some major challenges to using Syntex which prevents
libraries like Serde getting wide usage.  There are three major problems with
Syntex.  First, wiring Syntex into a project results in an inconvenient amount
of boilerplate code.  It requires the following `build.rs`, that is copy-pasted
into every Serde project, which registers the Serde plugin with Syntex, and
informs Syntex which files it should be expanding:

```rust
extern crate syntex;
extern crate serde_codegen;

use std::env;
use std::path::Path;

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();

    let src = Path::new("src/queen.rs.in");
    let dst = Path::new(&out_dir).join("queen.rs");

    let mut registry = syntex::Registry::new();

    serde_codegen::register(&mut registry);
    registry.expand("", &src, &dst).unwrap();
}
```

It also requires an unforntunate amount of macros to link in the generated
file, with a command like:

```rust
include!(concat!(env!("OUT_DIR"), "/queen.rs"));
```

Second, after a project has been Syntex-ified, it is actually inconvenient to
use in daily development because the generated files produce terrible error
messages.  This happens because error locations are reported inside the
generated file, not from within the template file.  Debugging an error then
requires opening up the generated file, finding the error, and then manually
searching the template file to find the error.

For example, a type error in `queen.rs.in` might produce this error message
that is in a file:

```
target/debug/build/test-ba65ec36dc6f8bb0/out/queen.rs:25:18: 2:23 error: mismatched types:
 expected `u64`,
    found `&'static str`
(expected u64,
    found &-ptr) [E0308]
target/debug/build/test-ba65ec36dc6f8bb0/out/queen.rs:25     let x: u64 = "foo";
                                                                          ^~~~~
```

Third, because of this difficulty with error locations, most users of Serde do
their development in Nightly Rust with the Serde plugin that is compatible with
Nightly Rust syntax extensions and gives good error locality.  Not only does
this cause more of our ecosystem to use Nightly Rust and it's unstable
features, it also requires even more inconvenient boilerplate code to make a
project compatible with Syntex and Nightly Rust plugins.  The `build.rs` from
before needs to be modified to:

```rust
#[cfg(feature = "with-syntex")]
mod with_syntex {
    extern crate syntex;
    extern crate serde_codegen;

    use std::env;
    use std::path::Path;

    pub fn main() {
        let out_dir = env::var_os("OUT_DIR").unwrap();

        let out_dir = env::var_os("OUT_DIR").unwrap();

        let src = Path::new("src/queen.rs.in");
        let dst = Path::new(&out_dir).join("queen.rs");

        let mut registry = syntex::Registry::new();

        serde_codegen::register(&mut registry);
        registry.expand("", &src, &dst).unwrap();
    }
}

#[cfg(not(feature = "with-syntex"))]
mod with_syntex {
    pub fn main() {}
}

pub fn main() {
    with_syntex::main();
}
```

and the entry point into the library needs to be modified to:

```rust
#![cfg_attr(not(feature = "with-syntex"), feature(custom_attribute, custom_derive, plugin))]
#![cfg_attr(not(feature = "with-syntex"), plugin(serde_macros))]

extern crate serde;

#[cfg(feature = "with-syntex")]
include!(concat!(env!("OUT_DIR"), "/lib.rs"));

#[cfg(not(feature = "with-syntex"))]
include!("lib.rs.in");
```

Beyond Syntex, there are a number of other tools that work by way of code
generation:

* [ANTLR](http://www.antlr.org/)
* [Lex](http://dinosaur.compilertools.net/lex/index.html)
* [Protocol Buffers](https://developers.google.com/protocol-buffers/)
* [Thrift](https://thrift.apache.org/)
* [Yacc](http://dinosaur.compilertools.net/yacc/index.html)

It is unlikely these projects would be rewritten in Rust, and so would also be
subject to the same "reporting errors in the generated file" that Syntex has.

---

# Detailed design
[design]: #detailed-design

This RFC proposes three changes that will help improve Rust's code generation
story.

## Source Mapping
[source mapping]: #source-mapping

Because of the challenges debugging generated code, this RFC proposes that Rust
be extended to produce and consume a file that contains a mapping from the
input generated file to the output Rust file.  Lets consider using the rustc
pretty printer to convert one Rust source into another.  For example, consider
a simple crate that's made up of two files.  `queen.rs`:

```rust
pub mod love;

pub struct Person { ... }
```

and it's submodule, `love.rs`:

```rust
use super::Person;

pub fn find(people: &[Person]) -> Option<&Person> {
    people.find(|person| person.lovable())
}
```

The pretty printer produces a single output file that merges the two files
together, and would look something like this:

```
pub mod love {
    use super::Person;

    pub fn find(people: &[Person]) -> Option<&Person> {
        people.find(|person| person.lovable())
    }
}

pub struct Person { ... }
```

By itself, this process loses the information that the module `love`
came from the file `love.rs`.  To avoid that, the pretty printer will
instead generate a file, `queen.rs.map`, that conceptually contains the
following mapping:

| dst line | dst col | source file | src line | src col | token           |
| -------- | ------- | ----------- | -------- | ------- | --------------  |
| 0        | 0       | "queen.rs"  | 0        | 0       | pub             |
| 0        | 4       | "queen.rs"  | 0        | 4       | mod             |
| 0        | 8       | "queen.rs"  | 0        | 8       | love\_canidates |
| 0        | 24      | "queen.rs"  | 0        | 24      | ;               |
| 2        | 0       | "love.rs"   | 0        | 0       | use             |

This mapping will then be used by the Rust compiler during parsing to map
tokens to their original location.

This approach draws much of it's inspiration from [JavaScript Source Maps
v3](...), which implements the same mechanism for translating arbitrary source
code into JavaScript.  This RFC proposes we adopt it's JSON format for our
mapping files for the lack of clear better choice.

Quoting from the from the Source Map Specification, this format looks like:

---

### Terminology

* Generated Code: The code which is generated by the compiler.
* Original Source: The source code whish has not been passed through the
	compiler.
* Base 64 VLQ: The VLQ is a Base64 value, where the most significant bit (the 6th bit)

### Example

```json
{
"version": 3,
"file": "out.rs",
"sourceRoot": "",
"sources": ["foo.rs.in", "bar.rs.in"],
"sourcesContent": [null, null, null],
"names": ["src", "maps", "are", "fun"],
"mappings": "A,AAAB;;ABCDE;"
}
```

* Line 1: The entire file is a single JSON object
* Line 2: File version (always the first entry in the object) and must be a
  positive integer.
* Line 3: An optional name of the generated code that this source map is associated with.
* Line 4: An optional source root, useful for removing repeated values in the
  "sources" entry.  This value is prepended to the individual entries in the
  "source" field.
* Line 5: A list of original sources used by the "mappings" entry.
* Line 6: An optional list of source content.  The contents are listed in the
  same order as the sources in line 5.
* Line 7: A list of symbol names used by the "mappings" entry.
* Line 8: A string with the encoded mapping data.

The "mappings" data is broken down as follows:

* each group representing a line in the generated file is seperated by a ";".
* each segment is separated by a ","
* each segment is made up of 1, 4, or 5 variable length fields.

The fields in each segment are:

1. The zero-based starting column of the line in the generated code that the
	 segment represents.  If this is the first field of the first segment, or the
	 first segment following a new generated line (";"), then this field holds
	 the whole base VLQ.  Otherwise, this field contains a base 64 VLQ that is
	 relative to the previous occurrence of this field.  *Note that this is
	 different than the fields below because the previous value is reset after
	 every generated line*
2. If present, an zero-based index into the "sources" list.  This field is a
	 base 64 VLQ relative to the previous occurrence of this field, unless this
	 is the first occurrence of this field, in which the whole value is
	 represented.
3. If present, the zero-based starting line in the original source represented.
	 This field is a base 64 VLQ relative to the previous occurrence of this
	 field, unless this if the first occurrence of this field, in which case the
	 whole value is represented.  Always present if there is a source field.
4. If present, the zero-based starting column of the line in the source
	 represented.  This field is a base 64 VLQ relative to the previous
	 occurrence of this field, unless this if the first occurrence of this field,
	 in which case the whole value is represented.  Always present if there is a
	 source field.
5. If present, the zero-based index into the "names" list associated with this
	 segment.  This field is a base 64 VLQ relative to the previous occurence of
	 this field, unless this is the first occurence of this field, in which case
	 the whole value is represented.

---

  There are some criticsms
with it's schema, such as it does not support easily extending source locations
with extra metadata.  However, the format does allow for extension points

The actual implementation is currently unspecified.




This would be generated into `queen.rs`:

```
set_span!("queen.rs.in", 1, 1);
pub mod love {
set_span!("love.rs.in", 1, 1);
    pub fn find(people: &[Person]) -> Option<&Person> {
        people.find(|person| person.lovable())
    }
}
set_span!("queen.rs.in", 2, 1);

pub struct Person { ... }
```

Where the arguments to `set_span!(...)` would be the source filename, the line,
and the column.  All this would allow a type error to be reported in the
original template:

```
queen.rs.in:2:18: 2:23 error: mismatched types:
 expected `u64`,
    found `&'static str`
(expected u64,
    found &-ptr) [E0308]
:queen.rs.in:2     let x: u64 = "foo";
                                ^~~~~
```

(Note: I'm not sure if the following paragraph is correct. It feels right, but
 I can't think of a concrete example for or against it)

While this uses a macro-like syntax, the simplest implementation would have
this macro be expanded during parse time, rather than the normal macro
expansion time.  This is because the `set_span!(...)` macro causes a
non-lexical state change in the parser.  All tokens following
`set_span!(...)` call will base their `Span`s off of the new location, until
either a new module is getting parsed, or another `set_span!(...)` is called.

## Source Search Paths
[paths]: #paths

In order to cut down on the boilerplate necessary including generated source into
a crate, the Rust Compiler should be extended to support the concept of source
search paths, similar to GCC's `-I some-path` option, as in
`rustc -I src -I $OUT_DIR/src`.  When Rust needs to look for some file, it will
check first in the current directory, then it will iterate through each search
path until the file is found.

The exceptions to this are the `#[path="..."]` and `include!(...)` and
related macros, which can only be used with an absolute path.  This is in order
to remain backwards compatible with current usage.

Cargo would then be updated to add the `$OUT_DIR` first in the search path
order.  This combined would allow users to use `mod queen;` instead of
`include!(...)`.

## Cargo Dependency Build Scripts

To cut down on the Cargo `build.rs` script boilerplate, this RFC proposes to
extend Cargo with the ability to execute build scripts provided by a
dependency.  This would enable code generators to use a more declarative
approach for configuration.  One possible `Cargo.toml` could look like:

```toml
...

[build-dependencies]
syntax = "*"
serde_codegen_syntex_plugin = "*"

[build]
dependency-scripts = [{ dependency = "syntex", script = "build-script" }]

[syntax-cargo-plugin]
plugins = ["serde_codegen_syntex_plugin"]
```

Where the project `syntex` would provide an executable `build-script` that
would get executed during the build phase.  This script then would re-parse the
`Cargo.toml` and extract out any other configuration options it needs.

# Drawbacks
[drawbacks]: #drawbacks

Source This would be one more macro, and could have a chance of a name
collision.  It would also add more complexity to the parser.  Some languages,
like Java, do not have `set_span!(...)` style functionality, and so it wouldn't
be unprecented to not have this functionality.

Adding support for Cargo dependency scripts could be very powerful beyond code
generators, such as enabling Cargo to easily integrate with other build
systems, but may also eventually lead us away from our simple, declarative
scripts we have now and take multiple steps towards plugin-heavy configuration
scripts.  It's possible to rewrite the `Syntex` boiler plate build script to be
simpler, which would be less painful to copy-paste everywhere.

# Alternatives
[alternatives]: #alternatives

* We could consider other names for the macros:
 * `#line "foo.rs" 1 2` in the style of CPP.
 * `set_line!("foo.rs", 1, 1);`
 * `set_source_span!("foo.rs", 1, 1);`
 * `set_location!("foo.rs", 1, 1);`

# Unresolved questions
[unresolved]: #unresolved-questions

* This macro will see little use outside of generated code, and doesn't necessarily
  need to be added to the default namespace. Could it instead be placed somewhere
  to be used with `#[macro_use]`?
* Is it actually backwards incompatible to have `#[path="..."]` find paths in
  the search paths?
* Do we need to track column information?
* Syntex uses a fork of `libsyntax`'s pretty printer to generate it's output,
	and it does a poor job generating similar input, even before expanding
	macros.  It's theoretically possible to generate a `set_line!(...)` call for
	each AST span to give correct information to the compiler, but this might balloon the
* Syntex uses a fork of `libsyntax`'s pretty printer to generate it's output,
	and it does a poor job generating similar input, even before expanding
	macros.  It's theoretically possible to generate a `set_line!(...)` call for
	each AST span to give correct information to the compiler, but this might
	balloon the output file.  Should we instead consider something like
	[JavaScript Source Maps](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)?
	Going this route would add a lot of complication to code generators.
