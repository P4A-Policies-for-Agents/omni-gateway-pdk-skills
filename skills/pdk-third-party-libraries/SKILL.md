---
name: pdk-third-party-libraries
description: Use when adding or troubleshooting third-party Rust dependencies in PDK policies, including Cargo.toml configuration, wasm32-wasip1 compatibility constraints, handling external service interactions via HttpClient, and the undocumented host foreign-function escape hatch (proxy-wasm call_foreign_function, e.g. FIPS crypto) with protobuf codegen and faux-mocked hostcalls.
---

# Skill: Using Third-Party Libraries

## Topic: Implementation

This skill covers how to include and use third-party Rust libraries in PDK custom policies.

## Overview

Proxy Wasm defines a low-level binary ABI that limits system calls. All third-party libraries must be compatible with the `wasm32-wasip1` Rust compilation target.

## Add a Library

Add the library to `Cargo.toml` dependencies:

```toml
[dependencies]
...
serde_urlencoded = "0.7.0"
```

Then use it in `lib.rs`:

```rust
serde_urlencoded::to_string([("token", "myToken")])
```

## Compatibility Notes

- Libraries that interact with external services (databases) or perform system calls (file I/O) are usually **not compatible** with `wasm32-wasip1`
- For external service interaction, use the PDK `HttpClient` instead
- Some libraries may compile but fail at runtime with errors like:
  - `Failed to load Wasm module due to a missing import: ...`
  - `Wasm VM failed to initialize Wasm code`
  - `Plugin configured to fail closed failed to load`
- In these cases, contact library owners or use a different library

## Host foreign-function calls — the escape hatch (undocumented / internal)

> **Caveat.** This is used by MuleSoft's own policies but is **not in the public PDK docs**. It
> reaches below the high-level `pdk` API into the proxy-wasm host ABI, so it is unsupported and can
> change without notice, and the specific host functions available (and their protobuf contracts)
> are gateway-version-dependent. Use only when no packaged PDK feature covers the need.

When a capability isn't wrapped by the high-level PDK API and can't be met by a `wasm32-wasip1`
crate (the constraints above), a policy can invoke an **Envoy host foreign function** directly. The
canonical case is FIPS-validated crypto: instead of bundling a Rust crypto crate, delegate the
digest to the host's FIPS-validated implementation. (For JWT specifically, prefer the packaged
`jwt-fips` Cargo feature — see [[pdk-cargo-features]] — over hand-rolling this.)

```rust
// Forwards to the host. `function_name` is a host-defined contract (e.g. "get_sha256_digest");
// arguments/return are protobuf-encoded bytes whose shape the host dictates.
pdk::classy::proxy_wasm::hostcalls::call_foreign_function(
    function_name,       // &str
    arguments,           // Option<&[u8]>
) // -> Result<Option<pdk::classy::proxy_wasm::types::Bytes>, pdk::classy::proxy_wasm::types::Status>
```

Two practical requirements observed in MuleSoft's `aws-sigv4` FIPS crypto module:

- **Protobuf codegen for the wire contract.** The host expects/returns protobuf messages; generate
  the Rust structs from a `.proto` in a `build.rs` (via `protobuf-codegen`) and `include!` them.
- **Make the hostcall mockable for unit tests.** Wrap `call_foreign_function` in a thin struct
  annotated with `#[cfg_attr(test, faux::create)]` / `#[cfg_attr(test, faux::methods)]` so tests can
  `faux::when!(mock.call("get_sha256_digest", _)).then(...)` and verify serialization, correctness
  (against RFC test vectors), and error handling without a live Envoy host. In production the wrapper
  is a zero-size forwarder to the real hostcall.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-libraries
- Example: Crypto Policy
- Host FFI pattern: MuleSoft-internal `microgateway-hybrid-policies` `libs/aws-sigv4/src/fips_crypto.rs`
  (undocumented; not from docs.mulesoft.com)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-libraries.adoc`
- **Snapshot:** 2026-09-19
- **Note:** the host foreign-function (`call_foreign_function`) section is undocumented — derived from
  MuleSoft's internal `microgateway-hybrid-policies`, not the docs page.
