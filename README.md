# Rust standard

A coding standard for Rust, and three Claude Code skills that review code with it.

- [standard/](standard/README.md) says how Rust code is organized, designed, written and documented. Its `rustfmt.toml` and `clippy.toml` apply the rules that tools can check.
- [skills/](skills/) holds the skills:
  - `review` looks for bugs and security holes. With `fix`, it fixes each one with a test.
  - `review-standard` checks Rust code against the standard. With `fix`, it makes the code follow it.
  - `review-architecture` reviews a crate's design, and proposes a plan to discuss before anything changes.

## Use it in a project

1. Copy `standard/` into the project, such as into `docs/standards/rust/`, and copy the folders of `skills/` into `.claude/skills/`.
2. Copy `standard/rustfmt.toml` to the workspace root. Its settings need nightly rustfmt (`cargo +nightly fmt`). The code itself builds on stable Rust.
3. Point Clippy at `clippy.toml`, in `.cargo/config.toml`:

   ```toml
   [env]
   CLIPPY_CONF_DIR = { value = "docs/standards/rust", relative = true, force = true }
   ```

4. Add the lints that implementation.md lists under "Panics" to the workspace's `Cargo.toml`, as `[workspace.lints.clippy]`.
5. Name the standard's folder in the project's `AGENTS.md` or `CLAUDE.md`, so that agents and the skills find it.

## Attribution

The standard is adapted from the [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html), by Google, licensed under [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/).
