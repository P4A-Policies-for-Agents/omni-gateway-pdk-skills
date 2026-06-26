---
name: pdk-upgrade-pdk
description: Use when upgrading PDK components including Anypoint CLI PDK plugin, PDK Rust libraries (pdk, pdk-test, pdk-unit), Anypoint Cargo plugin, policy Rust version, and WASI target migration from wasm32-wasi to wasm32-wasip1, with version-specific guidance for 1.8.0 resolver and metadata struct changes, and 1.9.0 FIPS flag, WebSocket support, and pdk-unit chunk size changes.
---

# Skill: Upgrading PDK

## Topic: Implementation

This skill covers how to upgrade the various components of the Omni Gateway Policy Development Kit (PDK) in a custom policy project.

## Overview

A PDK custom policy project has several separately versioned components that must be upgraded independently:

1. **Anypoint CLI PDK Plugin** - shared across all policies on the device
2. **PDK Rust Libraries** - per-policy, defined in `Cargo.toml`
3. **Anypoint Cargo Plugin** - per-policy, defined in the `Makefile`
4. **Policy Rust Version** - per-policy, defined in `Cargo.toml`
5. **WASI Crate** - per-policy, defined in the `Makefile` and test helpers

When you upgrade the Anypoint CLI PDK plugin, you must also upgrade the PDK Rust libraries and Anypoint Cargo plugin to compatible versions.

## Step 1: Upgrade the Anypoint CLI PDK Plugin

```bash
anypoint-cli-v4 plugins:install anypoint-pdk-plugin
```

Verify:

```bash
anypoint-cli-v4 plugins
```

**Note:** In PDK 1.7.0, the plugin was renamed from `anypoint-cli-pdk-plugin` to `anypoint-pdk-plugin`. If upgrading from earlier than 1.7.0, uninstall the old plugin first:

```bash
anypoint-cli-v4 plugins:uninstall anypoint-cli-pdk-plugin
anypoint-cli-v4 plugins:install anypoint-pdk-plugin
```

## Upgrading to 1.9.0: What's New

PDK 1.9.0 (released June 18 2026) is the current stable release. Upgrade to `1.9.0` for the `pdk`, `pdk-test`, and `pdk-unit` libraries, and for the `cargo-anypoint` plugin. Do not adopt the `1.9.1-alpha.2` prerelease.

Notable changes that may affect an upgrade:

- **`--fips` flag for `pdk create`:** New policy projects can be scaffolded as FIPS 140-3 compliant with `--fips`. This affects new projects only; existing projects are unchanged by the upgrade.
- **`AGENTS.md` in scaffolded projects:** Projects created with 1.9.0 now include an `AGENTS.md` file with proxy-wasm runtime rules, coding rules, and common pitfalls for AI coding agents. Existing projects do not gain this file automatically — copy it from a freshly scaffolded project if you want it.
- **WebSocket library (open beta):** PDK now provides `pdk::websockets` and FilterBuilder WebSocket hooks for writing policies on WebSocket APIs. No action needed unless you intend to use it.
- **`pdk-unit` default chunk size changed from 3 bytes to 30KB:** This aligns with Envoy and reduces event overhead. If any existing unit test relied on the old 3-byte fragmentation default, call `set_chunk_size(n)` explicitly to force the smaller chunk size for that test.

## Upgrading to 1.8.0: Known Issues

### Tokio Runtime Feature Compatibility Error

When using a Tokio runtime for testing, you may see a compilation error when building for WebAssembly:

```text
error: Only features sync,macros,io-util,rt,time are supported on wasm.
```

**Fix:** Add `resolver = "2"` under `[package]` in your policy's `Cargo.toml`. This enables the compiler to resolve dependencies separately for production code and test dependencies:

```toml
[package]
name = "my_policy"
version = "1.0.0"
rust-version = "1.88.0"
edition = "2018"
resolver = "2"
```

### Compilation Error When Constructing Metadata-Related Structs

PDK 1.8.0 marks metadata-related structs as `#[non_exhaustive]`, so you cannot construct them with struct expression syntax in tests. You will see:

```text
error[E0639]: cannot create non-exhaustive struct using struct expression
```

**Fix (option 1):** Use the new default constructor:

```rust
let mut metadata = FlexMetadata::default();
metadata.flex_name = "test_flex_name".to_string();
metadata.flex_version = "1.0.0".to_string();
```

**Fix (option 2):** Add the `non-exhaustive` crate as a dev-dependency:

```toml
[dev-dependencies]
non-exhaustive = "0.1.1"
```

Then use the macro:

```rust
use non_exhaustive::non_exhaustive;

let metadata = non_exhaustive!(FlexMetadata {
    flex_name: "test_flex_name".to_string(),
    flex_version: "1.0.0".to_string(),
});
```

## Step 2: Upgrade the PDK Rust Libraries

Update the `pdk` dependency and `pdk-test` dev-dependency versions in `Cargo.toml`. Both versions must match:

```toml
[dependencies]
pdk = { version = "1.9.0" }

[dev-dependencies]
pdk-test = { version = "1.9.0" }
```

**Note:** In versions earlier than 1.6.0, dependencies included `registry = "anypoint"`. When upgrading from pre-1.6.0, remove the `registry = "anypoint"` field since PDK libraries moved to [crates.io](https://crates.io/crates/pdk) in 1.6.0.

## Step 3: Upgrade the Anypoint Cargo Plugin

1. Open the `Makefile`
2. Find the `install-cargo-anypoint` target and update the version:

```makefile
install-cargo-anypoint:
	cargo install cargo-anypoint@1.9.0
```

3. Run:

```bash
make setup
```

4. Verify:

```bash
cargo anypoint --version
```

**Note:** When upgrading from pre-1.6.0, also remove `--registry anypoint --config .cargo/config.toml` flags from the `install-cargo-anypoint` target.

## Step 4: Upgrade the Policy's Rust Version

1. Open `Cargo.toml`
2. Update the `rust-version` field:

```toml
rust-version = "1.88.0"
```

Ensure the Rust version is installed on your device.

## Step 5: Upgrade the WASI Crate

Upgrade from `wasm32-wasi` to `wasm32-wasip1`:

1. In the `Makefile`, update the `TARGET`:

```makefile
TARGET                	:= wasm32-wasip1
```

2. In `tests/common/mod.rs`, update the `POLICY_DIR`:

```rust
pub const POLICY_DIR: &str = concat!(env!("CARGO_MANIFEST_DIR"), "/target/wasm32-wasip1/release");
```

## Upgrading from Pre-1.6.0 (Additional Steps)

When upgrading from versions earlier than 1.6.0, also clean up Anypoint Platform registry references in the `Makefile`:

1. Update the `setup` target to remove `registry-creds` and `login` dependencies:

```makefile
.PHONY: setup
setup: install-cargo-anypoint ## Setup all required tools to build
	cargo fetch
```

2. Delete the `login` and `registry-creds` targets from the `Makefile`.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-upgrade-pdk

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.9/modules/ROOT/pages/policies-pdk-upgrade-pdk.adoc`
- **Snapshot:** 2026-06-23
