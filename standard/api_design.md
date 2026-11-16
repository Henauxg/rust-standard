# API design

For library code these rules are binding, and for application code they are a
strong recommendation ([README.md](README.md#library-code-and-application-code)).

## Ownership and borrowing

### Avoid `Rc` and `Arc`

Prefer unique ownership and borrowing to `Rc` and `Arc`.

Use `Rc` or `Arc` when the ownership and borrowing structure without them
would be unreasonably complex to write, understand or maintain. This is a
judgment call for review.

Rust can usually represent an object graph without shared ownership. These
pointers add runtime cost and complexity of their own, and programmers coming
from other languages tend to reach for them too quickly.

### Prefer borrows to deep copies

Use `.clone()` when a deep copy is really needed. Do not use it to avoid
borrowing: design the code so it works with Rust's ownership and borrowing
rules. When the borrowing relationships become excessively complex, a
`.clone()` can be the pragmatic choice. Weigh the code's performance needs
when choosing.

This does not apply to cloning an `Rc` or an `Arc`, which copies the pointer,
not the data, and is the expected way to use them.

A deep copy allocates and copies data, which can be expensive.

## Function inputs and outputs

### Accept flexible references for immutable data

A function that borrows immutable data takes the most flexible reference type:
a reference to the data itself, not to a smart pointer, container or wrapper
around it.

| use | instead of | why |
| --- | --- | --- |
| `&T` | `&Box<T>` | borrows the data inside the box |
| `&[T]` | `&Vec<T>` | accepts any slice, not one kind of container |
| `&str` | `&String` | accepts any string slice, not only an owned string |
| `Option<&T>` | `&Option<T>` | borrows the data inside the `Option` |

Exception: non-linear character data, such as a rope.

The Clippy lints [`borrowed_box`](https://rust-lang.github.io/rust-clippy/master/index.html#borrowed_box)
and [`ptr_arg`](https://rust-lang.github.io/rust-clippy/master/index.html#ptr_arg)
check this. Callers can then pass data held in many ways without copying it
first.

### Return iterators rather than owned collections

Prefer returning an iterator (a concrete type or `impl Iterator`) to returning
a `Vec<T>`.

In application code, returning a `Vec<T>` is fine when it makes the code
simpler and the flexibility and performance of an iterator are not clearly
needed.

Exceptions, where a `Vec<T>` is the better return type:

- most callers need an owned vector anyway;
- the implementation naturally builds a vector;
- the result will likely be indexed at random;
- the algorithm works better as a `for` loop and the forced allocation matters
  little;
- the function's purpose is to create a new owned collection;
- the function returns a vector that already exists.

An iterator lets the caller choose how to consume the data: collect it, loop
over it, or chain more adapters, sometimes avoiding an allocation.

### Accept iterators rather than slices

Reusable library code that only iterates over an input accepts an iterator or
`impl IntoIterator`.

In application code, accepting a slice `&[T]` is fine when it makes the code
simpler and the flexibility of an iterator is not clearly needed.

An iterator parameter accepts data from any collection, does not require
copying it into contiguous memory, and works better in iterator chains.

### Accept `impl Into<T>` only where call-site ergonomics matter most

When library callers are likely to hold various types convertible to a `T`,
a function may accept `impl Into<T>` instead of `T`. Weigh this against code
size: it is not a goal to use `Into<T>` widely. Application code should almost
never take `impl Into<T>` parameters.

`impl Into<String>` accepts a `String` or a `&str` literal directly. The cost
is a generic signature: more complex, more code generated, and sometimes harder
type inference.

### Prefer an `enum` to a `bool` in APIs

Use an `enum` parameter instead of a `bool` when the parameter's purpose is not
obvious. Where a `bool` stays, consider a short inline comment at the call site
naming it.

## Errors

### Do not panic

Report a failure to the caller, never by a panic:

- an `Option` for a simple failure whose cause is obvious and about which the
  caller needs nothing more: a missing value, a mismatch;
- a `Result` for a failure the caller needs context to handle, such as an I/O
  or network error that correct code cannot rule out ahead of time, or bad data
  read from outside (a file, user input, the network).

```rust
// Good: the cause is obvious (overflow), and there is nothing more to say.
fn checked_add(a: u32, b: u32) -> Option<u32> { ... }

// Good: the caller needs to know why (not found, or permission denied).
fn open_file(path: &Path) -> Result<File, io::Error> { ... }
```

A panic is allowed only where no other way exists, decided case by case in
review, with a comment saying why. The typical case is an application that
cannot start because an invariant is badly broken, and even there an error
shown to the user is better than a panic.

Tests may panic: an assertion is how a test fails.

When a panic is allowed, write it with `expect` and a message
([implementation.md](implementation.md#prefer-expect-to-unwrap)).

### Use typed errors where callers tell failures apart

A function whose callers need to tell failures apart returns an error type of
its own: an `enum` per area of failure, written with
[`thiserror`](https://docs.rs/thiserror) or by hand. Library code always does.

[`anyhow`](https://docs.rs/anyhow) is for the top level of a binary, where an
error is only reported, and for tests.

## Types

### Give domain values their own types

Do not pass anonymous primitives or tuples where the value has a meaning in
the domain. Give the value a type.

```rust
// Bad
fn place(pos: (i32, i32), count: i32) {}

// Good
struct Pos { x: i32, y: i32 }
struct Count(u16);
fn place(pos: Pos, count: Count) {}
```

A struct with named fields is the default. A tuple struct with one field (a
newtype) is fine for a single wrapped value. A tuple with several fields is not
a type: `.0` and `.1` quickly become confusing. A short-lived tuple in a very
local context, where nothing can be mixed up, is fine.

Use a newtype rather than a type alias when:

- the type has a notably different meaning from the one it wraps;
- the type warrants an API of its own;
- the underlying type may change later, especially when not every use is ours;
- the values are easy to mix up. A date type may keep `Day`, `Month` and
  `Year` apart in its own calculations.

Do not invent wrappers for values that already have a strong type from the
language or a library, or for local counters, lengths and byte sizes that are
not domain values.

### Match a primitive's signedness and width to its value

The primitive inside a type matches the value, not a default `i32` or `i64`.

- Unsigned when the value cannot be negative: counts, sizes, extents, row IDs.
  Signed only when a negative value is a real, meaningful state.
- The smallest width that covers every possible value, with headroom that can
  be justified. Not `i64` or `u64` when `u32` or `u16` is enough.
- No `-1` or `0` as a sentinel to avoid an unsigned type. Use `Option<T>`.
- `usize` is for in-memory indexes and collection lengths, not for persisted
  IDs or values sent over the network.

### Give every persisted ID its own type

Every persisted ID gets its own newtype, one per table or kind of record.
Never reuse one ID type for another table, and never pass raw integers between
functions, components or storage helpers.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ItemId(u32);

impl ItemId {
    pub fn new(raw: u32) -> Self {
        Self(raw)
    }

    pub fn get(self) -> u32 {
        self.0
    }
}
```

Convert at the storage boundary only. With an SQL `INTEGER` column (signed),
the Rust API stays unsigned: `u32::try_from(row.id)` into `ItemId::new` when
reading, and `id.get().into()` as an `i64` when binding. Reject a negative or
overflowing ID. Never truncate one.

### Declare a reusable string once

A raw string used in more than one place is declared once, as a constant or
an enum, and every use refers to it. This holds for any such string. Some
examples: the words of a file format, a protocol or a command line (a key, a
kind or mode name, a file name, a flag), separators, sigils, operators, and
error codes. A typo in a repeated string still compiles, and a rename has to
find every copy.

A set of words that the code matches against or lists is an enum. Code
matches the enum, never the text, so a new word does not compile until every
`match` handles it. Derive the text, the parsing and the list of variants with
[strum](https://docs.rs/strum) rather than writing them by hand:

```rust
use strum::{Display, EnumString, IntoStaticStr, VariantArray};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Display, EnumString, IntoStaticStr, VariantArray)]
#[strum(serialize_all = "snake_case")]
enum KindName {
    Int,
    Bool,
    List,
}

// Bad
match text {
    "int" => int_kind(params),
    "list" => list_kind(params),
    _ => unknown(text),
}

// Good
match text.parse::<KindName>() {
    Ok(KindName::Int) => int_kind(params),
    Ok(KindName::List) => list_kind(params),
    Ok(KindName::Bool) => bool_kind(),
    Err(_) => unknown(text),
}
```

Any other reusable string, such as a file name or a separator, is a
constant: `const MANIFEST: &str = "manifest.json";`.

When the text comes from the variant names, renaming a variant changes the
format. Pin the words in a test, so that the rename fails it:

```rust
#[test]
fn words_are_pinned() {
    let words: Vec<&str> = KindName::VARIANTS.iter().map(|&kind| kind.into()).collect();
    assert_eq!(words, ["int", "bool", "list"]);
}
```

The same holds for the field names of a format another program reads: a file
saved to disk, a message to another process or to a script. Serialization
takes them from the struct's field names, so renaming a field changes the
format without any error. Pin them in a test that reads, or writes and
compares, a hand-written example of the format.

A message that names one of these words writes it through the enum or the
constant, so that a rename updates the message too:

```rust
// Bad
"`enum` needs `of` or `values`".to_owned()

// Good
format!("`{}` needs `{}` or `{}`", KindName::Enum, Key::Of, Key::Values)
```

This does not apply to the prose of a message, to a string used in one place
only, or to test input: a test's input, a fixture, a program that stands in
for another one in tests, and demo sample data. They are written as the user
or the other program would write them.

Class names and other values written in user interface markup may repeat, as
they do in the stylesheet that defines them. A class name the code computes or
matches is declared once.

An attribute cannot take a constant, so a word that an attribute must spell
out, such as a `serde` or `strum` rename, may repeat. A test pins these words,
so a typo fails it.

## Traits

### Add trait implementations when they become necessary

Low-level library code anticipates its users' needs. Application code adds a
trait implementation when it is needed.

### Implement standard library traits instead of custom methods

When a method matches the purpose and behavior of a standard library trait,
implement the trait: `impl FromStr for Foo`, not `fn from_str(&str) -> Self`
on `Foo`. Keeping the method as well, for discoverability, is fine.

The Clippy lint [`should_implement_trait`](https://rust-lang.github.io/rust-clippy/master/index.html#should_implement_trait)
lists the traits it checks.

### Derive what can be derived

Do not implement by hand what `#[derive()]` can generate.

### Prefer generics to `dyn Trait` in libraries

In library code, prefer a generic parameter (`T: Trait` or `impl Trait`) to a
`dyn Trait`, except where code size matters.

- A `dyn Trait` adds an indirection: a fat pointer, and a vtable lookup on
  every call. The compiler also cannot inline through it.
- Some traits cannot be used as `dyn` at all: those with generic methods,
  associated functions without `self`, or methods returning `Self`.

### Define a trait only for generic code or an extension

Define a trait only:

- to write generic code (`T: Trait` in bounds on functions and types);
- to accept or return an `impl Trait` or a `dyn Trait`;
- to define an extension trait (below).

Otherwise implement the functionality on the type directly, in an inherent
`impl`. Do not use traits merely to organize code.

### Name an extension trait with `Ext`, and own it with the type

An extension trait adds methods to a type from another crate. It is not meant
for generic code (`T: Trait`, `impl Trait`) or dynamic dispatch (`dyn Trait`).
Its name carries the suffix `Ext`: `ForeignTypeExt`.

In general, whoever owns the original type also owns its extension trait. The
exception is extension methods expected to become well known and valuable,
such as in a foundational library or a framework.

```rust
use foreign_crate::ForeignType;

trait ForeignTypeExt {
    fn additional_method(&self);
}

impl ForeignTypeExt for ForeignType {
    fn additional_method(&self) { ... }
}
```

An extension trait may also cover every type implementing a foreign trait,
which amounts to adding default methods to that trait:

```rust
use foreign_crate::ForeignTrait;

trait ForeignTraitExt {
    fn additional_method(&self);
}

impl<T: ForeignTrait> ForeignTraitExt for T {
    fn additional_method(&self) { ... }
}
```

This is about extension traits only. Defining an input trait (`DateLike`) and
implementing it on several foreign types is common and fine.

Owning the trait with the type avoids method names clashing between the type
and its extension, or between two extensions.

### Implement the important traits

- **A type with public fields:** consider implementing the traits all its
  fields implement.
- **An error type:** implement `Copy` whenever possible. A `Copy` error passes
  by value without moving ownership. It is not possible when the error holds a
  `String` or another non-`Copy` field.
- **`Copy`:** implement it when the type is reasonably sure to stay `Copy`.
- **`Clone`:** most types implement it, unless they represent an external
  resource.
- **`Default`:** implement it when the type has a `new()` or another default
  state (Clippy lint [`new_without_default`](https://rust-lang.github.io/rust-clippy/master/index.html#new_without_default)).
- **`PartialEq`, `Eq`:** for `==`, prefer implementing `PartialEq`. When
  implementing `Eq`, respect the reflexivity it requires.
- **`PartialOrd`, `Ord`:** for `<` and `>`, prefer implementing `PartialOrd`.
  When implementing `Ord`, respect the total order it requires.
- **A map key:** when the type is likely to be a map key or cached, implement
  `Hash` and `Eq`, and `Ord` too when it has a clear total order.

### Pair an important `From` with a constructor

When a `From` implementation is one of the main ways to build a type, add an
inherent `::new()`, `::new_from_foo()` or `::from_foo()` as well.

### Make an options struct implement `Default`

A struct with all-public fields that mainly bundles options for a function
implements `Default`, so callers can write `Options { a: 0, ..Default::default() }`.

## Naming

### Name methods as the standard library does

Follow the standard library's naming for methods, and the
[naming conventions](https://google.github.io/comprehensive-rust/idiomatic/foundations-api-design/predictable-api/naming-conventions.html)
of Comprehensive Rust. Use the prefixes `as_`, `to_` and `into_` with the
matching `self`
([API Guidelines](https://rust-lang.github.io/api-guidelines/naming.html#ad-hoc-conversions-follow-as_-to_-into_-conventions-c-conv),
Clippy lint [`wrong_self_convention`](https://rust-lang.github.io/rust-clippy/master/index.html#wrong_self_convention)).

### Name lifetimes descriptively

Give lifetimes descriptive names, especially lifetimes on types.

Exceptions:

- a lifetime on a very generic abstraction, such as `Cow`;
- the anonymous lifetime `'_`, wherever it works;
- single-letter lifetimes that only relate a method's input to its output, or
  that a local higher-ranked bound needs (`for<'a>`). Up to two such lifetimes
  are fine. Beyond two, use descriptive names.

### Name generic parameters after what they stand for

Use a short name, of one letter or a few, related to the kind of thing the
parameter stands for, ideally taken from its most important bound. `T` or `U`
fit a generic with few constraints, an API meant for almost any type, like
`Vec<T>`.

### Name constructors and conversions predictably

- A constructor is `new()`, or `new_with_foo()` when there are several. A
  constructor returning a `Result` is `try_new()`.
- A constructor that does work besides building the value, such as connecting,
  spawning, encoding or parsing, may be named after that work, as `File::open`
  and `TcpStream::connect` are.
- A value-to-value conversion is `from_foo()`, with a matching `From`
  implementation
  ([when a conversion is value-to-value](https://doc.rust-lang.org/std/convert/trait.From.html#when-to-implement-from)).

### Avoid abbreviations

Do not use abbreviations, acronyms above all, that someone outside the project
is unlikely to know. Do not abbreviate by dropping letters inside a word.

Exceptions:

- common abbreviations found on Wikipedia;
- abbreviations unlikely to confuse anyone outside the project, such as
  `IpAddr`. Author and reviewer decide. Even then, the preference is not to
  abbreviate.

## Accessors

### Name getters and setters by what they do

- `foo()`: a getter returning `&T`, or `T` by value when `T: Copy`;
- `foo_mut()`: a getter returning `&mut T`;
- `take_foo()`: returns the value and clears it, to `None` or another default;
- `set_foo()`: a setter taking a `T` by value;
- `replace_foo()`: a setter returning the old value;
- `with_foo()`, `with_mut_foo()`: an accessor through a closure, called as
  `object.with_foo(|foo| frob(foo))`.

```rust
impl Stuff {
    fn foo(&self) -> &Foo { ... }
    fn foo_mut(&mut self) -> &mut Foo { ... }

    /// Sets the "baz" property to `None` and returns the old value.
    fn take_baz(&mut self) -> Option<Baz> { ... }

    fn set_foo(&mut self, foo: Foo) { ... }

    /// Calls `f` with a borrow of the "bar" property and returns what `f` returns.
    fn with_bar<R>(&self, f: impl FnOnce(&Bar) -> R) -> R { ... }

    /// Calls `f` with a mutable borrow of the "bar" property and returns what `f` returns.
    fn with_mut_bar<R>(&self, f: impl FnOnce(&mut Bar) -> R) -> R { ... }

    /// Sets the "foo" property and returns the old value.
    fn replace_foo(&mut self, foo: Foo) -> Foo { ... }
}
```

An accessor through a closure suits two cases:

- the property sits in an interior mutability wrapper (`Mutex`, `RefCell`),
  which must limit access to a scope. The closure gives that scope, which is
  usually better than returning the guard (`MutexGuard<T>`), since the guard
  would expose an implementation detail;
- work or validation must follow a change, and a `set_foo()` cannot be used,
  for example because the property is expensive or impossible to clone.

A `with_foo()` usually lets the closure return a value and passes it on. With
two such accessors, one read-only and one mutable, name them `with_foo()` and
`with_mut_foo()`. With only one, `with_foo()` may also mutate.

### Implement expected API sets together

When implementing `len()`, implement `is_empty()` as well (Clippy lint
[`len_without_is_empty`](https://rust-lang.github.io/rust-clippy/master/index.html#len_without_is_empty)).

## Attributes and visibility

### Treat `#[doc(hidden)]` items of others as private

Do not use an item marked `#[doc(hidden)]` unless the code is ours.

### Mark a function `const` only if it will stay `const`

Mark a public function `const` only when it is expected to stay `const`, or
when we commit to helping its callers migrate if it changes. Turning a `const`
function into a regular one breaks its callers.

### Overload operators only for their mathematical meaning

Use an operator overload for its mathematical or logical purpose, never as a
shorthand to make code shorter.

### Mark library enums `#[non_exhaustive]`

An enum that is part of a library's public API is marked
`#[non_exhaustive]`, so a new variant does not break its callers.
Between crates of this repository it is not needed: one change fixes every
`match` (Clippy lint [`exhaustive_enums`](https://rust-lang.github.io/rust-clippy/master/index.html#exhaustive_enums)).

Exceptions, where the enum will not change:

- it describes an existing ontology, `enum Color { Red, Green, Blue }`;
- it has two states that could have been a `bool`,
  `enum Visibility { Visible, Hidden }`.

`#[non_exhaustive]` forces callers to add a wildcard arm (`_ => ...`) to their
`match`, so their code keeps compiling when a variant is added.

### Mark library structs with public fields `#[non_exhaustive]`

A struct with public fields that is part of a library's public API is marked
`#[non_exhaustive]`.

Exceptions:

- the fields are unlikely to change, because they describe an existing
  ontology (a rectangle) or the reviewer is otherwise confident;
- callers are meant to use struct update syntax
  (`Person { name: "foo", ..Default::default() }`), which `#[non_exhaustive]`
  forbids.

Adding a field, even a private one, to a struct whose fields are all public
breaks callers, who can no longer name every field in a struct literal. A
`#[non_exhaustive]` struct can be built with a literal only inside its own
crate, so private fields can be added without breaking anyone.

### Mark functions `#[must_use]` when ignoring the result is a mistake

Mark a function `#[must_use]` when:

- the caller cannot use it correctly while ignoring its return value,
  including a function without side effects, such as `HashMap::get()`;
- its return value reports an error or something else critical about what
  happened. A return type already marked `#[must_use]`, as `Result` is, needs
  nothing more.

### Mark types `#[must_use]` when they are returned to be used

Mark a type `#[must_use]` when the functions returning it would otherwise
need `#[must_use]` by the rule above:

- an error type, like `Result`;
- a lazy computation, which does nothing until its value is used.
