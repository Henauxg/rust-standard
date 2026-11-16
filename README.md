# Rust standard

A coding standard for Rust, and three Claude Code skills that review code with it.

- [standard/](standard/README.md) says how Rust code is organized, designed, written and documented. Its `rustfmt.toml` and `clippy.toml` apply the rules that tools can check.
- [skills/](skills/) holds the skills:
  - `review` looks for bugs and security holes. With `fix`, it fixes each one with a test.
  - `review-standard` checks Rust code against the standard. With `fix`, it makes the code follow it.
  - `review-architecture` reviews a crate's design, and proposes a plan to discuss before anything changes.

## Use it in a project

1. Copy `standard/` into the project, such as into `docs/standards/rust/`, and copy the folders of `skills/` into `.claude/skills/`. The commands in "Copy or update" below do it.
2. Copy `standard/rustfmt.toml` to the workspace root, or link to it. Its settings need nightly rustfmt (`cargo +nightly fmt`). The code itself builds on stable Rust.
3. Point Clippy at `clippy.toml`, in `.cargo/config.toml`:

   ```toml
   [env]
   CLIPPY_CONF_DIR = { value = "docs/standards/rust", relative = true, force = true }
   ```

4. Add the lints that implementation.md lists under "Panics" to the workspace's `Cargo.toml`, as `[workspace.lints.clippy]`.
5. Name the standard's folder in the project's `AGENTS.md` or `CLAUDE.md`, so that agents and the skills find it.

## Copy or update

Run these commands from the project's root, with a tag, a branch or a commit of this repository.
They work in sh and in PowerShell, and copy to the folders of step 1:

```sh
git fetch https://github.com/Henauxg/rust-standard <ref>
git rm -r -q -f --ignore-unmatch docs/standards/rust .claude/skills/review .claude/skills/review-standard .claude/skills/review-architecture
git read-tree --prefix=docs/standards/rust/ -u FETCH_HEAD:standard
git read-tree --prefix=.claude/skills/review/ -u FETCH_HEAD:skills/review
git read-tree --prefix=.claude/skills/review-standard/ -u FETCH_HEAD:skills/review-standard
git read-tree --prefix=.claude/skills/review-architecture/ -u FETCH_HEAD:skills/review-architecture
```

The copy is staged, ready to commit.
The next update replaces the copy whole, so make a change in this repository first.

## License

- The standard is under [CC BY 4.0](standard/LICENSE). It is adapted from the [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html), by Google, licensed under [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/).
- The skills are under the MIT license, in each skill's `LICENSE.txt`.

Each license file sits inside the folder it covers, so a copy of the folder keeps it.
