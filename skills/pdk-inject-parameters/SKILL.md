---
name: pdk-inject-parameters
description: Use when injecting parameters into PDK policy entrypoint and wrapped functions, including Configuration, Metadata, HttpClient, CacheBuilder, StreamProperties, PolicyViolations, RateLimitBuilder for entrypoint, and RequestState, ResponseState, RequestData, Authentication for wrapped functions with simplified function references.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-inject-parameters

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-inject-parameters.adoc`
- **Snapshot:** 2026-05-14
