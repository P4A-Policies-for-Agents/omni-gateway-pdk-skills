---
name: pdk-troubleshooting
description: Use when diagnosing PDK toolchain issues such as cargo-generate compilation errors on Mac, Anypoint CLI credential failures, group ID errors during policy creation, 401 authorization errors with outdated Rust, or split-model policy build failures.
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

**Fix:** This occurs when your Rust version is outdated. Ensure your Rust version meets the PDK requirements:

```bash
rustup update
```

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
