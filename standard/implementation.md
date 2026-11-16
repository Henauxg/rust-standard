# Implementation

## Panics

### Let Clippy check the no-panic rule

Panics are not used to report failures ([api_design.md](api_design.md#do-not-panic)).
A crate that follows this standard denies the lints that find them, and allows
them in tests:

```toml
# Cargo.toml
[lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
todo = "deny"
unimplemented = "deny"
unreachable = "deny"
indexing_slicing = "warn"
mod_module_files = "deny"
```

```toml
# clippy.toml, in this folder: a copy at the workspace root, or CLIPPY_CONF_DIR
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests = true
allow-indexing-slicing-in-tests = true
```

The test exemption covers `#[test]` functions and `#[cfg(test)]` modules, not
the helper functions of an integration test file under `tests/`. Such a file
allows the panic lints at its top, with the reason:

```rust
#![allow(
    clippy::panic,
    clippy::expect_used,
    clippy::unwrap_used,
    reason = "test code: a failure is a failed test"
)]
```

A panic that review allows carries its reason in an `expect` attribute, which
the compiler checks:

```rust
#[expect(clippy::expect_used, reason = "the rules table ships with the binary and is checked at startup")]
fn rules_table() -> RulesTable {
    RulesTable::parse(RULES).expect("the shipped rules table parses")
}
```

Indexing (`v[i]`) is only a warning: replacing every index with `.get(i)` is
heavy, and an index the surrounding code guarantees is fine. Prefer iterating
or `.get(i)` where the guarantee is not plain.

### Prefer pattern matching to check-then-unwrap

Do not rely on a runtime check to make a panicking call safe. Let a pattern
prove it to the compiler.

```rust
// Good: the match proves that `option` is `Some`, so no panic is possible.
if let Some(value) = option {
    use_value(value);
}

// Bad: the `unwrap` is correct only because of the check before it.
if option.is_some() {
    use_value(option.unwrap());
}
```

### Prefer `expect` to `unwrap`

Where a panic is allowed, write `expect` with a message saying why the value is
there, never `unwrap`.

### Check invariants with `debug_assert!`

`debug_assert!` may check an invariant the code already guarantees. It runs in
development builds and does nothing in release builds, so it finds bugs early
and never fails for a user. `assert!` outside tests is a panic, and follows the
panic rule.

### Give assertions a context string

Many assertions are clearer with a message saying why they should hold or what
broke them. Use the optional format string of `assert!` and `debug_assert!`.
Consider one especially:

- instead of a comment next to the assertion: the message appears in the
  output;
- when users may see the failure: a sentence helps them more than a Rust
  expression;
- when printing a value shows why it failed;
- to say whether a failure means the caller misused an API, or the
  implementation has a bug.

```rust
// Explains why the assertion holds.
debug_assert!(
    values.is_sorted(),
    "`values` comes from a `PriorityQueue`, so it is sorted"
);

// Prints the value that broke it.
assert!(refund > 0, "a refund must be positive, got {refund:?}");

// Flags a misuse of the API.
debug_assert!(self.stream.is_some(), "call connect() before executing queries");

// Bad: the message only repeats the condition.
assert!(values.is_sorted(), "values should always be sorted");
```

### Do not call `panic::catch_unwind`

Do not use `panic::catch_unwind`. It works only with `panic = "unwind"`, and
code that relies on it to recover hides the bug that panicked.

It is allowed as defense in depth, never relied on for any property: a
codebase that already avoids panics may wrap a critical entry point in it in
case one slips through. The author must convince the reviewer that the code
does most of what it can to avoid panics.

## Writing code

### Do not overuse type inference

Write enough explicit types that someone who knows the codebase can tell what
the inferred types are. Prefer a type on the variable to a turbofish. When the
compiler forces a turbofish, `_` inside it is fine. In a `let ... else`, a
turbofish is fine too: a type there would have to annotate the pattern.

Add a type in particular:

- on the result of a method with a generic return type, such as `.into()` or
  `.parse()`;
- on the result of a complex iterator chain.

### Match every variant of a `#[non_exhaustive]` enum

When matching a `#[non_exhaustive]` enum, list every variant. Variants that
need nothing go in the wildcard arm. When only some variants matter, use an
`if let` or a `let ... else`.

```rust
#[non_exhaustive]
pub enum Shape {
    Circle,
    Square,
    Triangle,
}

// Good
fn process_shape(shape: Shape) {
    match shape {
        Shape::Circle => println!("It's a circle"),
        Shape::Square => println!("It's a square"),
        // Triangle and any later variant need nothing.
        Shape::Triangle | _ => {}
    }
}

// Good
fn process_shape(shape: Shape) {
    if let Shape::Circle = shape {
        println!("It's a circle");
    }
}

// Bad
fn process_shape(shape: Shape) {
    match shape {
        Shape::Circle => println!("It's a circle"),
        _ => {}
    }
}
```

Exception: only some variants matter, and a `match` reads more clearly than an
`if let`.

### Prefer a `for` loop to an unclear combinator chain

- Avoid long, complex combinator chains.
- Do not build a chain in the header of a `for` loop. Assign it to a variable,
  whose name says what it is, and loop over the variable.
- Avoid `Iterator::for_each` without a good reason. A `for` loop usually reads
  better.

A combinator is better for a short chain, and where it holds logic the loop
would have to write out.

```rust
// Bad: conditions and outside state inside a chain.
let mut error_count = 0;
let _processed_items: Vec<_> = items
    .into_iter()
    .map(transformation_that_may_fail)
    .map(|result| {
        if result.is_err() {
            error_count += 1;
        }
        result
    })
    .filter_map(Result::ok)
    .collect();

// Good
let mut error_count = 0;
let mut processed_items = Vec::with_capacity(items.len());

for item in items {
    match transformation_that_may_fail(item) {
        Ok(val) => processed_items.push(val),
        Err(_) => error_count += 1,
    }
}
```

```rust
// Bad: a chain in the loop's header.
for bug in bug_descriptions
    .iter()
    .filter_map(|description| parse_bug_description(description).ok())
    .filter(|bug| bug.priority <= Priority::P1)
{
    assign_bug(bug);
}

// Good
let urgent_bugs = bug_descriptions
    .iter()
    .filter_map(|description| parse_bug_description(description).ok())
    .filter(|bug| bug.priority <= Priority::P1);

for bug in urgent_bugs {
    assign_bug(bug);
}
```

```rust
// Good: the combinator holds the logic.
let threshold_exceeded = values.iter().any(|&v| v > threshold);

// Bad: the loop rewrites what `any()` does.
let mut threshold_exceeded = false;
for v in values {
    if v > threshold {
        threshold_exceeded = true;
        break;
    }
}
```

### Build strings with `format!` and `write!`

Use `format!()` or `write!()`:

- to join more than two strings: `format!("{a}{b}{c}")`, not `a + b + c`;
- to substitute values into a literal string, instead of `str::replace`.

They allocate less than `+`, and standard format strings beat a templating
syntax of our own.

### Do not run code before `main()`

Do not make code run before `main()`, for example with the
[`ctor`](https://docs.rs/ctor/latest/ctor/) crate.

## Concurrency

### Never read through a poisoned mutex

Never reach inside a poisoned mutex to get at its content, however tempting
the case.

Exception: code whose work must still run after a panic, such as starting
and stopping a child process, may read through the poison when two things hold:

- the content stays valid whatever a panicking holder left, such as an enum
  whose every value is a state the code handles;
- the path must run all the same, because skipping it costs more than the
  panic did, such as processes left running after the program ends.

A comment at each such read says why both hold there.

## Macros

### Prefer plain Rust to macros

To express an API, prefer, in this order:

1. ordinary Rust functions, types, traits and modules;
2. `macro_rules!` macros;
3. other code generation, such as a build script;
4. procedural macros.

When choosing a macro or generated code, weigh its costs:

- editors, code search and other tools work less well on code using macros;
- macros create dialects of the language, which make it harder to move
  across the codebase, and a reader can assume less about a macro call than
  about ordinary code;
- refactorings across the codebase become harder and more expensive.

A `macro_rules!` macro works on the syntax tree, so it is less fragile than
textual code generation, which makes it the choice for simple generation.

## Unsafe code

### Write unsafe code only when safe code cannot do it

Do not write unsafe code unless there is no way to do the same with safe code.
When unsafe code is needed:

- wrap it in a safe abstraction, kept apart from the rest of the logic, ideally
  in its own crate with few dependencies beyond the standard library;
- add a comment starting with `SAFETY:` that explains why the code is correct;
- justify it technically, for one block or for a whole file or crate. The
  justification answers: why is the code not written safely? If the reason is
  performance, where are the benchmarks?

## Unstable features

### Use stable Rust

Do not use unstable language or library features. Formatting with nightly
rustfmt ([organization.md](organization.md#formatting)) does not count: it is
a tool, and the code still builds on stable.

## Tests

### Use the built-in test harness

Tests use the built-in `#[test]`. Unit tests sit in a `mod tests` at the end of
the file they test, with `use super::*`. Integration tests sit in the crate's
`tests/` directory.

### Fuzz targets follow the test rules

A fuzz target feeds random input to a function and reports a bug by panicking,
as a test does. A crate's fuzz targets sit in a `fuzz/` crate beside it,
outside the workspace, because `cargo-fuzz` runs on nightly Rust. Their code
still builds on stable: nightly is a tool here, as it is for rustfmt.

The workspace's lints do not reach a crate outside it, so the fuzz crate's
`Cargo.toml` copies the workspace's `[lints.clippy]` table. Lint it with
`cargo clippy` run in its folder. Each target allows the panic lints at its
top, as an integration test file does:

```rust
#![no_main]
#![allow(
    clippy::panic,
    clippy::expect_used,
    clippy::unwrap_used,
    reason = "a fuzz target reports a bug by panicking"
)]
```
