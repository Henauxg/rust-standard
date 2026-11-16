# Rust coding standard

This is the standard for Rust code in this project: how code is organized,
designed, written and documented. "Style" here covers more than formatting. It
includes API design and implementation choices.

It is our own adaptation of the
[Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html).
The rules specific to Chromium and Fuchsia are left out, the flat file layout
rule is replaced, and our own rules are added (strong types in
[api_design.md](api_design.md)).

| file | covers |
| --- | --- |
| [organization.md](organization.md) | crates, modules and files, imports, formatting |
| [api_design.md](api_design.md) | ownership, function inputs and outputs, errors, types, traits, naming, accessors, attributes |
| [implementation.md](implementation.md) | implementation concerns, concurrency, macros, unsafe code, unstable features, tests |
| [documentation.md](documentation.md) | doc comments and code comments |

## Goals

Most rules come from one or more of these goals.

- **Readability and maintainability come before ease of writing.** Code lives
  for a long time, so a little more effort when writing is worth it when it
  makes the code easier to read later.
- **Rules favor automation.** Formatters, linters and refactoring tools work
  better on consistent code. A refactoring that a tool applies across the whole
  codebase needs code the tool can handle easily, so some rules choose patterns
  that refactor well.
- **Rules stay consistent with the wider Rust ecosystem where possible.** The
  ecosystem is already fairly consistent. Following it means fewer surprises and
  less to learn.
- **Every rule pulls its weight.** Every author and reviewer has to keep these
  rules in mind, so a rule is included only when it has a clear benefit.
- **Rules settle minor disagreements.** Reasonable people disagree on many small
  style questions. A rule saves rehashing them in every review. Rules that have
  no other benefit are kept to a minimum.

## Built on the public Rust rules

Follow the official public Rust rules, except where this standard says
otherwise:

- the [Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/), most
  of which rustfmt applies automatically ([organization.md](organization.md#formatting));
- the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/).

A conflict is a case where this standard asks for a pattern those rules
discourage, or one that goes against the pattern they mainly encourage.

## Library code and application code

Rules about API design apply differently to two kinds of code.

- **Library code** has callers outside this repository: users of a published
  crate, or other projects. Its callers cannot be fixed in the same change as
  the library, so for library code the API design rules are binding.
- **Application code** is everything else, however many crates of this
  repository use it: one change can update it and all its callers. For
  application code, the API design rules are a strong recommendation.
  Application code should favor simplicity and directness over broad reuse:
  avoid generalizing early, and build what the immediate need requires.

## License

This standard comes from [rust-standard](https://github.com/Henauxg/rust-standard),
under [CC BY 4.0](LICENSE).

It is adapted from the [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html),
by Google, licensed under
[CC BY 3.0](https://creativecommons.org/licenses/by/3.0/).
