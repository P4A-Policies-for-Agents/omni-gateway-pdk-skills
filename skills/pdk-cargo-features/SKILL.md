---
name: pdk-cargo-features
description: Use when configuring PDK Cargo feature flags in Cargo.toml to shrink compiled policy WASM — disabling the default jwt or xml_validator libraries, or enabling the FIPS-compliant jwt-fips JWT backend (mutually exclusive with jwt). Applies to PDK 1.9.1+.
---

# Skill: Configuring PDK Cargo Features

## Topic: Implementation

Since PDK 1.9.1 the JWT and XML Validator libraries are gated behind Cargo features so you can exclude them from the compiled policy WASM when a policy does not use them. Trimming unused features reduces WASM size. This skill covers the feature flags and the `Cargo.toml` syntax.

## Feature Flags

| Feature | Default | Provides |
|---------|---------|----------|
| `jwt` | on | JWT library (`pdk::jwt`) — native RustCrypto backend |
| `xml_validator` | on | XML Validator library (`pdk::xml_validator`) |
| `jwt-fips` | off | FIPS-compliant JWT backend; delegates cryptography to the proxy-wasm host instead of bundling native RustCrypto |

The default feature set is `default = ["jwt", "xml_validator"]`. An in-place version bump to 1.9.1+ changes nothing — both libraries stay enabled unless you opt out.

**Note the underscore:** the feature is `xml_validator` (underscore), not `xml-validator`.

## Trim Unused Libraries

Set `default-features = false` on the `pdk` dependency, then re-add only what you use:

```toml
# Disable both JWT and XML Validator:
pdk = { version = "1.9.2", default-features = false }

# Keep only JWT:
pdk = { version = "1.9.2", default-features = false, features = ["jwt"] }

# Keep only XML Validator:
pdk = { version = "1.9.2", default-features = false, features = ["xml_validator"] }
```

If you leave the default `pdk = { version = "1.9.2" }`, both features are on.

## FIPS-Compliant JWT (`jwt-fips`)

`jwt-fips` is **mutually exclusive** with the default `jwt` backend, so you must disable default features when using it (otherwise both JWT backends are pulled in and the build fails):

```toml
# FIPS-compliant JWT only:
pdk = { version = "1.9.2", default-features = false, features = ["jwt-fips"] }

# FIPS-compliant JWT with XML Validator:
pdk = { version = "1.9.2", default-features = false, features = ["jwt-fips", "xml_validator"] }
```

The JWT API surface (`SignatureValidator`, `JwtGenerator`, claim accessors) is the same under `jwt-fips` — see the `pdk-jwt` skill. Only the crypto backend differs. For scaffolding a fully FIPS-compliant project use `pdk create --fips`; see `pdk-upgrade-pdk`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Enabling `jwt-fips` without `default-features = false` | Conflicts with default `jwt`; always disable defaults first. |
| Writing `xml-validator` (hyphen) | The feature is `xml_validator` (underscore). |
| Expecting a version bump to 1.9.1+ to shrink WASM automatically | Defaults stay on; you must explicitly `default-features = false`. |
| Using JWT/XML APIs after disabling their feature | Re-add the feature, or the `use pdk::jwt::*` / `pdk::xml_validator` import won't resolve. |

## Documentation Reference

- Configure features: https://docs.mulesoft.com/pdk/latest/policies-pdk-create-project#configure-pdk-features
- FIPS JWT: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-jwt

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.9/modules/ROOT/pages/policies-pdk-create-project.adoc` (#configure-pdk-features)
- **Release notes:** https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.9.1)
- **Snapshot:** 2026-07-28
