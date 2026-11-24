---
name: pdk-troubleshooting
description: Use when diagnosing PDK toolchain issues such as cargo-generate compilation errors on Mac, Anypoint CLI credential failures, group ID errors during policy creation, 401 authorization errors with outdated Rust (PDK 1.11 needs 1.97+), compilation errors from a stale Cargo.lock after a PDK upgrade, rust-lld or duplicate `_initialize` link errors, or split-model policy build failures.
---

# Skill: PDK Troubleshooting

## Overview

This skill covers common issues encountered when using Omni Gateway Policy Development Kit (PDK) and how to resolve them. For debugging custom policy logic, see the `pdk-debug-local` and `pdk-test-locally` skills.

## Common Issues

### Cargo-Generate Error on a Mac Device

When compiling Rust on a Mac device for the first time, you may see:

```
$ cargo install cargo-generate
error: linking with `cc` failed: exit status: 1
```

**Fix:**

1. Install Xcode command line tools:

```bash
xcode-select --install
```

2. If the error persists after installing Xcode, create or edit `~/.cargo/config.toml`:

```toml
[target.x86_64-apple-darwin]
rustflags = [
  "-C", "link-arg=-undefined",
  "-C", "link-arg=dynamic_lookup",
]
[target.aarch64-apple-darwin]
rustflags = [
  "-C", "link-arg=-undefined",
  "-C", "link-arg=dynamic_lookup",
]
```

### Anypoint CLI Credentials Error

Using any PDK command without Connected App credentials results in:

```
Error: Failed to launch the browser process! undefined
[...]:ERROR:ozone_platform_x11.cc(239)] Missing X server or $DISPLAY
```

**Fix:** Anypoint CLI requires multi-factor authentication (MFA) with a Connected App. Configure Anypoint CLI with MFA authentication using a Connected App.

### Group ID Error When Creating the Policy Project

The `ANYPOINT_ORG` environment variable can use either the organization name or ID. If set to the organization name, the policy creation command fails to infer the group ID.

**Fix:** Use the organization **ID** (not name) for `ANYPOINT_ORG`. If you cannot change the variable, either:

- Enter the group ID when prompted:
  ```
  Please provide a valid group-id (the id of the organization that will own the asset):
  ```

- Use the `--group-id` flag:
  ```bash
  anypoint-cli-v4 pdk policy-project create -n <policy-name> --group-id <organization-id>
  ```

### 401 Authorization Error When Running `make setup`

```
Caused by:
  failed to get successful HTTP response from `https://anypoint.mulesoft.com/crates[...]/download` [...], got 401
make: *** [install-cargo-anypoint] Error 101
```

**Fix:** This occurs when your Rust version is outdated. PDK 1.11.0 requires **Rust 1.97.0 or later**
(PDK 1.10 required 1.88.0). Update the toolchain and the policy's `rust-version`:

```bash
rustup update
rustc --version   # must report 1.97.0 or later for PDK 1.11
```

If the repo pins a toolchain in `rust-toolchain.toml`, bump `channel` there too. `rustup update` does
not override a pinned channel.

### Compilation Error After Upgrading PDK

After bumping `pdk`, the build fails inside a transitive dependency (commonly `jwt-simple` pulled by
`pdk-jwt-lib`) because the existing `Cargo.lock` pins a version incompatible with the new library.
PDK 1.11.0 also updated third-party dependencies for security advisories, and an old lockfile keeps
the vulnerable versions.

**Fix:** refresh the lockfile, then rebuild:

```bash
cargo update                                   # or: cargo update -p jwt-simple --precise <version>
# alternatively delete Cargo.lock and let Cargo regenerate it
make build
```

### Unit Tests Fail to Link on x86_64 Linux (Rust 1.90+)

Since Rust 1.90, `x86_64-unknown-linux-gnu` links with `rust-lld` by default. Unit tests of
non-PDK `proxy-wasm` policies (for example, ones using `pdk-unit` with the `proxy-wasm-rust-sdk`
feature) can fail at load time with `symbol lookup error` on unresolved host calls.

**Fix:** opt out of `lld` for the host target in `.cargo/config.toml`:

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "linker-features=-lld"]
```

### `duplicate symbol: _initialize` When Building for `wasm32-wasip1`

A rustc newer than the one the project was validated with can clash with `crt1-reactor.o` and fail
the WASM link with `duplicate symbol: _initialize`.

**Fix:** pin the validated toolchain in `rust-toolchain.toml` (for example `channel = "1.97.0"`) and
make CI scripts honor the pin instead of forcing `stable`.

### Error: No such file or directory When Running `make build` (Split Model Policy)

```
Error: No such file or directory (os error 2)
make: *** [build] Error 1
```

**Fix:** Ensure you have run the following commands **in order** before building:

1. `make setup`
2. `make release-local`
3. Publish the definition asset version
4. Then run `make build`

### Error Looking for the Specified Asset When Running `make build-asset-files` (Split Model Policy)

```
Error: There was a problem looking for the specified asset.
make: *** [build-asset-files] Error 2
```

**Fix:** Same as above — ensure you have run `make setup` and `make release-local`, and that the definition asset version has been published before building the implementation.

## See Also

- Skill: Debugging Custom Policies with the PDK Debugging Playground (`pdk-debug-local`)
- Skill: Test PDK Policy Locally (`pdk-test-locally`)
- Skill: Upgrading PDK (`pdk-upgrade-pdk`)

## Source Ref

- **Docs:** https://docs.mulesoft.com/pdk/latest/policies-pdk-troubleshooting (PDK 1.11)
- **Release notes:** https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.11.0)
- **Snapshot:** 2026-11-24
