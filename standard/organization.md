# Code organization and formatting

## Terms

- A **crate** is a unit of compilation. It always contains one nameless
  module, the root module.
- A **module** organizes code inside a crate and controls the visibility of
  names. Modules nest, and they can split code across files and directories.
- A **leaf module** has no submodules, apart from `mod tests` and binding
  modules for another language (such as `#[cxx::bridge]`).

## Crates

### Keep each crate focused

A crate focuses on one abstraction. Split it when two parts have no reason to
be in the same crate. A crate is still allowed to grow large when the trait
coherence rules force several types and traits into one library, or when it is
a foundational library.

The crate root may re-export the public API of its leaf modules, so that its
submodules are an implementation detail users do not need to know.

Decide first whether the APIs belong in the same crate at all. Only then decide
whether submodules would help organize them.

## Modules and files

### Name a module with submodules `foo.rs`, beside `foo/`

A module `foo` that has submodules lives in `foo.rs`, and its submodules in the
directory `foo/`. Do not use `foo/mod.rs`. The lint `clippy::mod_module_files`
enforces this.

```
src/
  lib.rs
  formula.rs          mod lexer; mod parser; pub use parser::parse;
  formula/
    lexer.rs
    parser.rs
```

This is the layout the Rust Book presents since the 2018 edition. A project
using `mod.rs` ends up with many files of the same name, which are hard to tell
apart in an editor.

### Name a file and a leaf module after its main item

When a leaf module has one main user-defined item, such as `Foo`, name the
module and its file after it: `mod foo` in `foo.rs`. A leaf module that is not
mainly about one abstraction may be named freely.

This shows a file's purpose at a glance.

### Do not implement APIs in non-leaf modules

A non-leaf module contains only module declarations (not definitions) and
re-exports. With the layout above, `foo.rs` holds the `mod` lines and the
`pub use` lines, and the code lives in `foo/`.

A small amount of code that ties the submodules together is fine. The goal is
that these modules hold little code.

This makes things easier to find, and keeps structure apart from content.

## Imports

### Do not use glob imports

Do not use glob imports, except in these cases, where they are allowed:

- `use super::*` in an inline module, such as `mod tests`;
- `use MyEnum::*` inside the methods of that enum, and in functions mainly
  about it;
- `pub use submodule::*` in a non-leaf module.

A third-party crate's prelude may be imported with a glob, such as
`use bevy::prelude::*`. Do not define preludes of our own.

Glob imports make it harder to see where a name comes from when reading code
without editor navigation, and harder for tools to refactor.

### Import one module per `use` statement

A `use` statement has no `::` inside its braces. rustfmt applies this with
`imports_granularity = "Module"` (see [Formatting](#formatting)).

```rust
// Good
use foo::Foo;
use foo::bar::Bar;
use foo::bar::Baz;

// Good
use foo::Foo;
use foo::bar::{Bar, Baz};

// Bad
use foo::{Foo, bar::Bar};
```

### Import traits where they are used

Import a trait inside the function that uses it, unless the trait's methods
are used throughout the file.

```rust
fn foo(s: &str) -> Result<u8, std::num::ParseIntError> {
    // The trait is imported in the function, not at module scope.
    use std::str::FromStr;

    u8::from_str(s)
}
```

### Call a top-level function through its module

To call a top-level function of another module or crate, import the module
and call `module::function()`.

```rust
// Good
use std::mem;

let x = mem::transmute(y);

// Bad
use std::mem::transmute;

let x = transmute(y);
```

Exceptions:

- functions whose names are already unambiguous on their own;
- modules whose name is likely to collide with another, where importing the
  function reads more clearly.

Many Rust codebases assume an unqualified lowercase name is local, so `foo()`
is read as coming from the same module. The rule mainly settles review
disagreements, and gives a small edge when reading code without editor
navigation.

### Import type names and use them unqualified

Refer to types and traits without their module path: write `Regex`, not
`regex::Regex`, with `use regex::Regex` at the top of the file or in the
function.

Exceptions:

- where names are ambiguous, refer to a common name through its module
  (`io::Error`), or rename it (`use std::io::Error as IoError`);
- a large group of related types, such as the
  [`hir` module of rustc](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/hir/index.html);
- macros and generated code, where requiring a `use` gets in the way;
- code with many dependencies, where importing everything would make a very
  long import block for little benefit.

The rule mainly settles review disagreements.

## Formatting

### Format with rustfmt

All code is formatted with rustfmt, from the nightly toolchain, so that every
setting below applies. The code itself still builds on stable Rust.

```sh
cargo +nightly fmt
```

The settings are in [`rustfmt.toml`](rustfmt.toml), in this folder. A
crate that follows the standard uses them, by a copy at the workspace root or
with `--config-path`:

```toml
max_width = 100
hex_literal_case = "Lower"
format_macro_matchers = true
imports_granularity = "Module"
```

- `max_width`: at most 100 columns per line.
- `hex_literal_case`: hex literals in lowercase, `0xab123cdef` rather than
  `0xAB123CDEF`.
- `format_macro_matchers`: macro matchers are formatted too.
- `imports_granularity`: one module per `use` statement (above).

rustfmt does not cover one more rule: write an invisible character in a
literal as an escape. A zero-width joiner is `\u{200D}`.

rustfmt never splits a string literal, so a long one runs past the width.
Leave it whole: a message kept in one piece is found by searching the code
for the text a user saw. Do not split it with `\` or `concat!`.

### Opt out of formatting only for alignment or generated code

`#[rustfmt::skip]` is allowed only for:

- **manual alignment**, where lines must line up for a reason specific to
  them: several similar lines kept aligned to be read together, or a function
  building a matrix;
- **generated code**, though generating it at build time is better.

Never skip formatting for a personal style preference.
