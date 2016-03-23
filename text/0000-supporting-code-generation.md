- Feature Name: `code_generation`
- Start Date: 2016-02-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a series of changes to the Rust compiler and Cargo in order
to better support code generators. It proposes the following high level
changes:

* Add a macro to improve error messages that can be used in generated code to
  inform the compiler where in the template file an error may have occured.
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

## Error location
[error location]: #error-location

The compiler should be extended to allow generated code to inform the following
code was generated in a separate template file.  For example, presume we had a
crate `queen.rs.in`,

```rust
mod love_candidates;

pub struct Person { ... }
```

that contained a submodule, `love_candidates.rs`:

```rust
use super::Person;

pub fn find(people: &[Person]) -> Option<&Person> {
    people.find(|person| person.lovable())
}
```

This would be generated into `queen.rs`:

```
set_span!("queen.rs.in", 1, 1);
pub mod love_candidates {
set_span!("love_candidates.rs.in", 1, 1);
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
