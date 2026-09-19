---
name: pdk-inject-parameters
description: Use when injecting parameters into PDK policy entrypoint and wrapped functions, including Configuration, Metadata, HttpClient, CacheBuilder, StreamProperties, PolicyViolations, RateLimitBuilder for entrypoint, and RequestState, ResponseState, RequestData, Authentication for wrapped functions with simplified function references, plus the undocumented internal LdapBuilder (pdk::ldap) and ReadinessInstance (pdk::metrics::readiness) injectables.
---

# Skill: Injecting Parameters

## Topic: Implementation

This skill covers how to inject parameters into PDK policy entrypoint and wrapped functions.

## Entrypoint Injectable Parameters

The following parameters can be injected into the `#[entrypoint]` configuration function:

- `Configuration`: Policy configuration parameters from schema definition
- `Metadata`: Metadata about policy, Omni Gateway instance, API instance, and Anypoint Organization
- `HttpClient`: Enables HTTP calls from the policy
- `CacheBuilder`: Provides caching features
- `StreamProperties`: Share properties with other policies processing the same request
- `PolicyViolations`: Report policy violations for monitoring dashboards
- `RateLimitBuilder`: Rate limiting functionality to control request rates

## Wrapped Function Injectable Parameters

The `on_request` and `on_response` wrapped functions accept:

- `HttpClient`: HTTP calls
- `StreamProperties`: Share properties between policies
- `RequestState`: Access request headers and body (only in `on_request`)
- `ResponseState`: Access response headers and body (only in `on_response`)
- `RequestData`: Share data between request and response functions (only in `on_response`)
- `Authentication`: Read/share authentication data with other policies

**Best practice:** Inject parameters only where needed. If initialization is required only once, inject into `#[entrypoint]` for performance.

## Simplified Function References

If wrapped functions only receive injectable parameters, the lambda is not required:

```rust
// Instead of:
let filter = on_request(|request_state| request_filter(request_state))
    .on_response(|response_state, request_data| response_filter(response_state, request_data));

// Use:
let filter = on_request(request_filter)
    .on_response(response_filter);
```

## Passing Additional Parameters

For non-injectable parameters (e.g., config), define a lambda and pass references:

```rust
#[entrypoint]
async fn configure(launcher: Launcher, Configuration(bytes): Configuration) -> Result<()> {
    let config = serde_json::from_slice(&bytes)?;
    let tuple: (u32, u32) = (10, 10);

    let filter = on_request(|request_state| request_filter(request_state, &config, &tuple));

    launcher.launch(filter).await?;
    Ok(())
}
```

## Undocumented injectables (internal — use at your own risk)

> **Caveat.** The two injectables below are used by MuleSoft's own production policies but are **not
> in the public PDK docs**. Treat them as unsupported/unstable: they can change or be removed
> without notice, and are absent from the official injectable list above. Documented here because
> they were observed in shipping MuleSoft policies; prefer a documented alternative when one exists.

### `LdapBuilder` — LDAP directory client (`pdk::ldap`)

Injects a builder for authenticating credentials against an LDAP directory. Inject into
`#[entrypoint]`, configure once, and pass the built `LdapClient` by reference into the request filter.

```rust
use pdk::ldap::{LdapBuilder, LdapClient, LdapError};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    ldap_builder: LdapBuilder,
    policy_violations: PolicyViolations,
) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes)?;
    let ldap: LdapClient = ldap_builder
        .new()
        .server_url(config.ldap_server_url)
        .server_user_dn(config.ldap_server_user_dn)
        .server_user_password(config.ldap_server_user_password)
        .search_base(config.ldap_search_base)
        .search_filter(config.ldap_search_filter)
        .build()?;

    launcher.launch(on_request(move |rs, auth| request_filter(rs, &ldap, auth))).await?;
    Ok(())
}

// In the filter: ldap.authenticate_encoded(&credential).await returns Result<_, LdapError>
// (LdapError::AuthenticationFailed / RequestFailed / ServerError).
```

### `ReadinessInstance` — defer the "ready" signal until async init completes (`pdk::metrics::readiness`)

Injects a readiness handle so a policy that needs async startup work (fetch JWKS, validate contracts)
can hold off signaling readiness until that work succeeds, rather than reporting ready the moment
`configure` returns.

```rust
use pdk::metrics::readiness::{Readiness, ReadinessInstance};

#[entrypoint]
async fn configure(launcher: Launcher, readiness: ReadinessInstance, /* ... */) -> Result<()> {
    // ... kick off async init (JWKS fetch, contract load) ...
    // Call readiness.ready() exactly once, only after init has actually completed:
    readiness.ready();
    // Guard against calling it twice (track a `readiness_sent` bool in your state).
    launcher.launch(/* ... */).await?;
    Ok(())
}
```

See [[pdk-runtime-model]] for why control-plane-independent, resilient startup matters.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-inject-parameters

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-inject-parameters.adoc`
- **Snapshot:** 2026-09-19
- **Note:** the `LdapBuilder` and `ReadinessInstance` injectables are undocumented — observed in
  MuleSoft's internal `microgateway-policies` (`ldap_authentication`, `jwt_validation`,
  `oauth2_token_introspection`), not on the public docs page.
