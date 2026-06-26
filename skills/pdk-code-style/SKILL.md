---
name: pdk-code-style
description: Use when formatting PDK policy Rust code following the official Rust Style Guide with project-specific overrides for naming conventions, imports, constants, comments, functions, structs, traits, generics, control flow, and Cargo.toml configuration.
---

# Skill: PDK Code Style

## Topic: Code Style

This skill defines the Rust code style rules to follow when writing PDK policy code. Based on the official [Rust Style Guide](https://doc.rust-lang.org/style-guide/), with **project-specific overrides** for PDK policy projects (marked with **[PROJECT]**).

For project-specific coding patterns and best practices (error handling, body modification, SSE streaming, JSON-RPC, metrics, etc.), see the **pdk-coding-best-practices** skill.

## Formatting Basics

- Use **spaces**, not tabs
- Each indentation level: **4 spaces**
- Maximum line width: **100 characters**
- Prefer **block indent** over visual indent
- Use **trailing commas** in multi-line comma-separated lists
- Separate items with zero or one blank line
- No trailing whitespace

```rust
// Block indent (preferred)
a_function_call(
    foo,
    bar,
);

// Visual indent (avoid)
a_function_call(foo,
                bar);
```

## Naming Conventions

| Kind | Convention | Example |
|---|---|---|
| Types | `UpperCamelCase` | `PolicyConfig` |
| Enum variants | `UpperCamelCase` | `Action::Allow` |
| Struct fields | `snake_case` | `max_retries` |
| Functions/methods | `snake_case` | `request_filter` |
| Local variables | `snake_case` | `auth_token` |
| Macros | `snake_case` | `my_macro!` |
| Constants / statics | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |

For reserved words, use raw identifiers (`r#crate`) or trailing underscore (`crate_`). Don't misspell (`krate`).

### Constants [PROJECT]

String constants use explicit `&'static str` (not just `&str`):

```rust
pub const POST_METHOD: &'static str = "POST";
pub const CONTENT_TYPE_HEADER: &'static str = "content-type";
const CLIENT_ENTITY_NAME: &str = "Client"; // private constants may omit 'static
```

Use `pub(crate)` for constants shared within the crate but not exported:

```rust
pub(crate) const SOURCE_ADDRESS: &[&str] = &["source", "address"];
```

## Comments

- Prefer line comments (`//`) over block comments (`/* */`)
- Put a single space after `//`
- Prefer comments on their own line; if after code, put a single space before `//`
- Comments should be complete sentences (capital letter, period)
- Comment-only lines: max **80 characters** (including `//`, excluding indentation), or max line width, whichever is smaller
- Prefer outer doc comments (`///`) over block doc comments (`/** */`)
- Use inner doc comments (`//!`) only for module-level or crate-level docs
- Put doc comments **before** attributes

```rust
/// A well-documented function.
///
/// Returns the computed value.
#[inline]
fn compute(x: i32) -> i32 {
    // Validate the input range.
    assert!(x > 0);
    x * 2
}
```

## Functions

```rust
// Single-line signature
fn foo(arg1: i32, arg2: i32) -> i32 {
    ...
}

// Multi-line signature — break after `(`, before `)`, trailing comma
fn foo(
    arg1: i32,
    arg2: i32,
) -> i32 {
    ...
}
```

Ordering: `[pub] [unsafe] [extern ["ABI"]] fn name(...)`

## Structs, Enums, Unions

```rust
struct Foo {
    a: A,
    b: B,
}

enum FooBar {
    First(u32),
    Second,
    Error {
        err: Box<Error>,
        line: u32,
    },
}

// Small struct variant on one line (spaces around braces, no trailing comma)
enum FooBar {
    Error { err: Box<Error>, line: u32 },
}
```

Prefer unit struct (`struct Foo;`) over empty struct (`struct Foo {}` or `struct Foo();`).

## Traits and Impls

```rust
// Empty — single line
trait Foo {}
impl Foo {}

// Non-empty — break after `{`, before `}`
pub trait Bar {
    fn method(&self);
}

impl Bar for Foo {
    fn method(&self) { ... }
}

// Bounds
trait Foo: Debug + Bar {}
```

## Generics and Where Clauses

```rust
// Single line
fn foo<T: Display, U: Debug>(x: Vec<T>, y: Vec<U>) -> ...

// Multi-line generics — trailing comma
fn foo<
    T: Display,
    U: Debug,
>(x: Vec<T>, y: Vec<U>) -> ...

// Where clause
fn function<T, U>(args)
where
    T: Bound,
    U: AnotherBound,
{
    body
}
```

Prefer single-letter names for generic parameters. Prefer inline bounds for short constraints; use `where` for longer ones.

## Imports

```rust
use a::b::c;
use a::b::{foo, bar, baz};
```

- No spaces around braces in imports
- Version-sort within groups; `self` and `super` come first
- Don't merge or re-order across blank lines
- Normalize: `use a::self;` -> `use a;`; `use a::{b};` -> `use a::b;`

### Import Grouping [PROJECT]

Group imports with blank lines between groups, in this order:

1. `mod` declarations (first)
2. External crate imports (`anyhow`, `serde`, `serde_json`, `mime`, etc.)
3. PDK imports (`pdk::*`)
4. Local crate imports (`crate::*`, `super::*`)

```rust
mod generated;

use anyhow::Result;
use serde_json::from_slice;

use pdk::hl::*;
use pdk::logger;

use crate::generated::config::Config;
use crate::http_utils::{CONTENT_TYPE_HEADER, POST_METHOD};
```

### Wildcard Imports [PROJECT]

`use pdk::hl::*;` is the standard import for PDK high-level APIs. Wildcard imports are acceptable for the PDK HL module specifically.

## Let Statements

```rust
let pattern: Type = expr;

// If too long, break after `=`
let pattern: Type =
    expr;

// let-else — single line if small
let Some(1) = opt else { return };

// Multi-line let-else
let Some(1) = opt else {
    return;
};
```

## Control Flow

```rust
// No extraneous parentheses
if condition {
    ...
} else if other {
    ...
} else {
    ...
}

// Single-line if-else in expression position (if small)
let y = if x { 0 } else { 1 };

// Multi-line condition — opening brace on its own line
while let Some(foo)
    = a_long_expression
{
    ...
}
```

## Match

```rust
match foo {
    foo => bar,
    a_very_long_pattern | another_pattern => {
        no_room_for_this_expression()
    }
    foo => {
        let a = statement();
        an_expression()
    }
    bar => {}
}
```

- Block-indent arms once
- Trailing comma if not using a block
- Never start a pattern with `|`
- Never break after `=>` without using a block body

## Function Calls

```rust
// Single-line
foo(x, y, z)

// Multi-line — each arg on its own line, trailing comma
a_function_call(
    arg1,
    a_nested_call(a, b),
)
```

## Method Chains

```rust
// Single line if small
x.foo().bar().baz(x, y, z);

// Multi-line — break before `.`, after `?`
let foo = bar
    .baz?
    .qux();
```

## Closures

```rust
|arg1, arg2| expr

move |arg1: i32, arg2: i32| -> i32 {
    expr1;
    expr2
}

// Combine with function call
foo(first_arg, x, |param| {
    action();
    foo(param)
})
```

Use `{}` when there's a return type, statements, comments, or multi-line control flow.

## Blocks

```rust
// Empty block
{}

// Single-line block (expression position only)
let _ = { a_call() };
let _ = unsafe { a_call() };

// Multi-line block
let foo = {
    a_call_inside_a_block();
    the_value
};
```

Keyword before block (`unsafe`, `async`) on same line with a space.

## Types

- `&'a T`, `&mut T` — no space after `&`
- `*const T`, `*mut T` — no space after `*`
- `[T; expr]` — space after `;`
- `(A, B, C)` — spaces after commas
- `Foo::Bar<T, U>` — spaces after commas, no spaces around `<>`
- `T + U + V` — spaces around `+`
- Break before `+` when multi-line

## Attributes

```rust
#[repr(C)]
#[foo(foo, bar)]
#[long_multi_line_attribute(
    split,
    across,
    lines,
)]
struct CRepr {
    x: f32,
    y: f32,
}
```

- One attribute per line
- Single `derive` attribute (combine multiple into one)
- Space around `=` in attributes: `#[foo = 42]`

## Expressions

- Prefer expression-oriented style: `let x = if y { 1 } else { 0 };`
- No spaces in ranges: `0..10`, `x..=y`
- Spaces around binary operators: `x + 1`
- No space for unary operators: `!x`, but `&mut x`
- Use parentheses for clarity, not relying on precedence

## Cargo.toml

- Same indentation (4 spaces) and line width (100 chars) as Rust code
- Blank line between sections, not within sections
- `[package]` at top; `name` and `version` first, `description` last
- Version-sort keys within sections
- Use bare keys (no quotes for standard keys)
- Single space around `=`

```toml
[dependencies]
crate1 = { path = "crate1", version = "1.2.3" }

[dependencies.extremely_long_crate_name]
path = "extremely_long_path"
version = "4.5.6"
```

### Cargo.toml Conventions [PROJECT]

Policies in this repo are **standalone crates** — pin dependency versions directly rather than via a workspace. Keep version strings consistent across policies so upgrades stay coordinated:

```toml
[dependencies]
pdk = { version = "1.9.0", features = ["enable_stop_iteration"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = { version = "1.0", default-features = false, features = ["alloc", "raw_value"] }
anyhow = "1.0"

[dev-dependencies]
pdk-test = { version = "1.9.0" }
pdk-unit = { version = "1.9.0" }
httpmock = "0.6"
reqwest = { version = "0.11", features = ["json"] }
```

Section ordering: `[package]` -> `[lib]` -> `[package.metadata.anypoint]` -> `[dependencies]` -> `[dev-dependencies]`.

The `[lib]` section specifies `crate-type = ["cdylib"]` for WASM policy output.

## Documentation Reference

- Source: https://doc.rust-lang.org/style-guide/
- GitHub: https://github.com/rust-lang/rust/tree/HEAD/src/doc/style-guide/
- Snapshot: 2026-03-27
