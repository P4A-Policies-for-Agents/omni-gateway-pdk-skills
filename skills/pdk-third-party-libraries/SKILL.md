---
name: pdk-third-party-libraries
description: Use when adding or troubleshooting third-party Rust dependencies in PDK policies, including Cargo.toml configuration, wasm32-wasip1 compatibility constraints, and handling external service interactions via HttpClient.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-libraries
- Example: Crypto Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-libraries.adoc`
- **Snapshot:** 2026-05-14
