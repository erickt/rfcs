- Feature Name: macro_line_control
- Start Date: 2016-02-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add a macro that adjusts the current span to improve error reporting in code
generated files.  This would allow a code generator that takes an input file
`main.rs.in` with an error like this:

```rust
fn main() {
    let x: u64 = "foo";
    ...
}
```

which generates a file `$OUT_DIR/main.rs` like this:

```rust
struct GeneratedCode {
    ...
}

set_source_span!("main.rs.in", 1, 1);
fn main() {
    let x: u64 = "foo";
    ...
}
```

That the type error would be reported as this:

```
main.rs.in:2:18: 2:23 error: mismatched types:
 expected `u64`,
    found `&'static str`
(expected u64,
    found &-ptr) [E0308]
:main.rs.in:2     let x: u64 = "foo";
                               ^~~~~
```

Instead of like this:

```
target/debug/build/test-ba65ec36dc6f8bb0/out/main.rs:25:18: 2:23 error: mismatched types:
 expected `u64`,
    found `&'static str`
(expected u64,
    found &-ptr) [E0308]
target/debug/build/test-ba65ec36dc6f8bb0/out/main.rs:25     let x: u64 = "foo";
                                                                         ^~~~~
```

# Motivation
[motivation]: #motivation

[Syntex](https://github.com/serde-rs/syntex) is a convenient tool that enables
libraries like [Serde](https://github.com/serde-rs/serde) to support
plugin-like functionality while the plugin functionality is stabilized. It
works by parsing some code, expands the syntax extensions, and generates an
output file that is then actually compiled by Rust. Unfortunately, one of the
downsides of Syntex is that errors will be reported as if they came from the
generated file, and not from the original source file. This means that the
typical development work flow is that people will develop against Rust Nightly
and the `serde_macros` plugin for good error messages, and then test with Rust
Stable to just make sure things work before publishing. This is obviously not
ideal since we'd prefer most people to stay on Stable to make sure people don't
get too attached to Nightly features.

Beyond Syntex, there are a number of other tools that work by way of code
generation:

* [ANTLR](http://www.antlr.org/)
* [Lex](http://dinosaur.compilertools.net/lex/index.html)
* [Protocol Buffers](https://developers.google.com/protocol-buffers/)
* [Thrift](https://thrift.apache.org/)
* [Yacc](http://dinosaur.compilertools.net/yacc/index.html)

These code generators are not written in Rust and therefore are unlikely to
integrate directly with the Rust compiler.

# Detailed design
[design]: #detailed-design

`set_source_span!` will be implemented with a macro that re-spans tokens
that exist in the current file following the call.


Macros are implemented by way of a pass that is run after the parser has
completed.  In order to implement `set_source_span!`, 


At the time of writing this RFC, the Rust compiler tracks the position in the
source files of every token in a program.  Each source file is loaded into a
`CodeMap`, which builds a mapping from integer byte positions to the actual
position in a file.  These byte positions are then stored as a `Span` attached
to each Token.

There is currently 
The order that 


This RFC proposes that a built-in macro, `set_source_span!(...)`, to Rust that
allows some source file to inform the parser that the next block of code
originated from a different file than the current one being processed.  This
macro would take three arguments: the original file, line, and the column that
the following code after the macro.

This proposal is not a new concept, and similar functionality exists in other
languages.  Both
[C Preprocessor](https://gcc.gnu.org/onlinedocs/cpp/Line-Control.html) and
[C#](https://www.google.com/search?q=c%23+%23line&ie=utf-8&oe=utf-8)
use the `#line 1 foo.c` directive, whereas O'Caml uses a slight variant
`# 1 "foo.ml"`.


This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# Drawbacks
[drawbacks]: #drawbacks

This would be one more macro, and could have a chance of a name collision.  It
would also add more complexity to the parser.


# Alternatives
[alternatives]: #alternatives

* We could not implement this, and live with the incorrect spans in generated
  source.  For example, Java doesn't support this functionality unless you
  write JVM assembly, so it wouldn't be unprecedented.
* We could use the same syntax as CPP and have `#line 1 foo.rs`, which would
  make it easier on existing code generators.  However, I feel that this would
  not integrate well with the rest of the language.
* We could consider other names for the macros:
 * `set_line!("foo.rs", 1, 1);`
 * `set_span!("foo.rs", 1, 1);`
 * `set_location!("foo.rs", 1, 1);`

# Unresolved questions
[unresolved]: #unresolved-questions

* This macro will see little use outside of generated code, and doesn't necessarily
  need to be added to the default namespace. Could it instead be placed somewhere
  to be used with `#[macro_use]`?
