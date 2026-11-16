---
name: review-standard
description: Review Rust code against the Rust coding standard (the project's copy, or the one beside this skill), for a pull request, a branch, the working tree or a path. With the `fix` argument, fix what does not comply, in passes, until the code complies. Use when asked to review Rust changes against the standard, check compliance, or make code compliant.
---

# Review Rust code against the coding standard

This skill checks that Rust code follows the coding standard, and with `fix`
makes the code follow it. Whether the code works is out of scope: `/review`
checks it.

## Where the standard is

The standard is a folder that holds `README.md`, `organization.md`,
`api_design.md`, `implementation.md`, `documentation.md`, `clippy.toml` and
`rustfmt.toml`. Use the project's copy when it has one: its `AGENTS.md` or
`CLAUDE.md` names it, or search the repository for `api_design.md`. Otherwise
use the `standard/` folder that comes with this skill, at `../../standard/`
from the skill's own folder. Below, `<standard>` is the folder's path.

## Arguments

`/review-standard [fix] [target]`

- `fix`: fix what does not comply, in passes (below). Without it, only report.
- `target`, what to review:
  - nothing: the current branch's changes, both committed since it left the
    default branch and uncommitted, untracked `.rs` files included;
  - a pull request number or URL: its changes against its base;
  - a branch or a commit range: `main..feature`;
  - a path (a file, a directory, a crate): all of its code, not only changes.

## What is judged

- Rust code: `.rs` files, and the `[lints]` of the `Cargo.toml` of a touched
  crate.
- For changes, only the changed lines and the items they touch. The code
  around them is context. A violation elsewhere in the same file is not
  reported. For a path, everything in it is judged.
- A deviation the code already justifies is respected: an
  `#[expect(lint, reason = "...")]`, or a comment giving the decision. Check
  that the reason holds.

## 1. Read the standard

Read every Markdown file in `<standard>`: `README.md`, `organization.md`,
`api_design.md`, `implementation.md` and `documentation.md`. Every finding
cites the rule by its file and heading.

For each crate touched, decide whether it is library code or application code
(`README.md`, "Library code and application code"). Library code has callers
outside this repository, such as the users of a published crate. When
unsure, ask the user. The API design rules are binding for library code, and
recommendations for application code. Report them as such.

## 2. Collect what to review

- Nothing given: find the default branch (`git remote show origin`, else
  `main`), take `git merge-base HEAD origin/<default>`, and review
  `git diff <merge-base>` plus untracked `.rs` files.
- A pull request: fetch its head (`git fetch origin pull/<n>/head`) and diff
  against its base, or use the host's pull request tools.
- A branch or a range: `git diff <range>`.
- A path: every `.rs` file under it.

List the crates touched: the nearest `Cargo.toml` above each file. A fuzz
crate (`<crate>/fuzz/`) sits outside the workspace: run the tools below in its
folder, without `-p`, and check that its `[lints.clippy]` table matches the
workspace's `[workspace.lints.clippy]`.

## 3. Run the tools

Use quiet flags, so the output stays short. For each touched crate:

1. **Formatting:**

   ```sh
   cargo +nightly fmt -p <crate> -- --check --config-path <standard>/rustfmt.toml
   ```

   If nightly rustfmt is missing, install it with
   `rustup component add --toolchain nightly rustfmt`. If that fails, say that
   formatting was not checked.

2. **Lints:**

   ```sh
   CLIPPY_CONF_DIR=<standard> cargo clippy -q -p <crate> --all-targets -- \
     -D clippy::unwrap_used -D clippy::expect_used -D clippy::panic -D clippy::todo \
     -D clippy::unimplemented -D clippy::unreachable -W clippy::indexing_slicing \
     -D clippy::mod_module_files
   ```

   Add `-D missing_docs` for a library crate. `CLIPPY_CONF_DIR` points Clippy
   at the standard's `clippy.toml`, which exempts tests from the panic lints.

For changes, keep only the diagnostics on changed lines.

## 4. Read the code against the other rules

Tools check little of the standard. Read the code with each rule in mind, at
least:

- **organization.md:** focused crates; `foo.rs` beside `foo/`; files named
  after their main item; no code in non-leaf modules; no glob imports beyond
  the allowed ones; one module per `use`; trait imports where used; top-level
  functions called through their module; type names unqualified; string
  literals never split to fit the width.
- **api_design.md:** borrows over clones, `Rc` and `Arc`; flexible
  references; iterators or collections; `impl Into`; an `enum` over a `bool`;
  no panics, including `assert!` and indexing outside tests; typed errors and
  `anyhow`; strong types (anonymous primitives and tuples, signedness and
  width, sentinels, persisted IDs); reusable strings declared once (an enum
  derived with strum for a set of words, pinned by a test; a constant
  otherwise; messages naming the words through them); traits (standard traits, `derive`,
  generics over `dyn`, a trait only for generics or an extension, the
  important traits, `From` with a constructor, `Default` on options); naming
  (standard library style, lifetimes, generic parameters, constructors,
  abbreviations); accessors; `#[non_exhaustive]` and `#[must_use]` on library
  APIs; `const` functions; operator overloads.
- **implementation.md:** pattern matching over check-then-unwrap; `expect`
  messages; `debug_assert!` for invariants; assertion messages; explicit types
  where inference hides them; matching every variant of a `#[non_exhaustive]`
  enum; `for` loops over unclear combinator chains; `format!` and `write!`; no
  `catch_unwind`; no code before `main()`; poisoned mutexes; macros; unsafe
  code with `SAFETY:`; stable Rust; test layout; fuzz targets.
- **documentation.md:** doc comments (every public item of library code, and
  in application code only where they add something); their quality; comments
  only for a constraint, a measurement or a trap; present tense, no history;
  length; plain writing; `TODO(#issue)`.

## 5. Report

One entry per finding, grouped by file:

```
crates/core/src/rules.rs:42  api_design.md, "Do not panic"
  `table.get(key).unwrap()` panics when the key is missing.
  Fix: return `Option<&Rule>`, and let the caller decide.
```

Mark a finding that is a recommendation (an API design rule on application
code). End with the number of findings per file of the standard, and list the
findings that need a decision (below) on their own.

When nothing is found, say so, and say what was checked: the tools run, and
the files read.

## Fix mode: passes

With `fix`, repeat:

1. **Review** the current state of the target (steps 3 and 4).
2. **Stop** if the pass finds nothing: the code complies.
3. **Fix** every finding that does not need a decision, with the smallest
   change that applies the rule. Change nothing the standard does not ask
   for. A fix may reach beyond the target only where it must, such as the
   callers of a changed signature inside this repository. Run the formatter
   last:

   ```sh
   cargo +nightly fmt -p <crate> -- --config-path <standard>/rustfmt.toml
   ```

4. **Verify** that the code still works: `cargo check -q` and `cargo test -q`
   on each touched crate. If something broke, fix it before the next pass.
5. **Go on** to the next pass, which also judges the fixes themselves.

Stop after five passes, or when a finding comes back after a fix, and report
what is left and why. Do not commit: the caller decides.

## Findings that need a decision

Report these, and do not fix them:

- a panic that may be the allowed exception, where no other way exists:
  propose the `#[expect(..., reason = "...")]` form or the alternative;
- a change to a library crate's public API, which breaks callers outside the
  repository;
- a rule broken on purpose without a stated reason, where a reason may exist;
- two rules in conflict, or a case the standard does not settle: propose a
  rule when one would help.

At the end, ask the user about each of them, with a recommendation.

## Final report in fix mode

- the passes run, and in each the fixes made (file and line, rule);
- what is left, and why;
- the commands run and their results: formatting, Clippy, `cargo check`,
  `cargo test`.
