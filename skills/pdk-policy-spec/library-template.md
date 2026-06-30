# <lib_name>

<One paragraph: the problem this crate solves and the policies that depend on it. Match the H1 to
the crate name.>

## What it provides

<The contract, as narrative — not a rustdoc dump. Cover:>

- **The problem it solves** and the mental model a caller needs.
- **The load-bearing public surface** — the key types, traits, and functions, each in backticks
  (e.g. `RetryCache`, `parse_claims`, `BodyHandler`). Don't enumerate every item; `cargo doc` is
  the authoritative API reference.
- **What it deliberately does NOT solve** — the boundaries, so callers don't reach for it for the
  wrong job.

### <Capability>

<One subsection per distinct capability the crate offers, if it has more than one. Describe the
capability and how it's meant to be used.>

## Examples

<Short Rust snippets — 3 to 15 lines each, one per common use case. Policies are the integration
tests; don't reproduce them here.>

```rust
// minimal usage of the primary entry point
```

```rust
// a second common use case
```
