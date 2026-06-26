---
name: pdk-experimental-feature
description: Use when working with undocumented PDK experimental features (experimental_enable_stop_iteration, experimental_metrics, experimental), including mandatory code annotations with feature name and expected GA version, Cargo.toml feature flag documentation, and migration checklists for when features graduate to stable.
---

# Skill: Using PDK Experimental Features

## Topic: Implementation

This skill covers guidelines for using experimental features in the MuleSoft PDK (Policy Development Kit). Experimental features are undocumented, intended for internal usage, and may have breaking changes at GA.

## Important Disclaimer

The PDK team does **not** document experimental features. They are intended for internal usage and early validation only. Experimental features:

- **Are not covered by official documentation** — the only way to understand them is by reading the PDK source code in the Cargo registry (`~/.cargo/registry/src/`)
- **May have breaking changes at GA** — the API surface, behavior, and feature flag names can change without notice when the feature graduates to stable
- **Are not supported** — issues encountered with experimental features may not receive the same level of support as stable features

## Mandatory Code Annotations

All usage of experimental features in policy code **must** be properly flagged with:

1. **The experimental feature name** being used
2. **The expected GA version** where the feature will become stable
3. **A tracking marker** so the experimental usage can be found and replaced when GA lands

### Cargo.toml

When enabling an experimental feature, add a comment documenting the feature, the expected GA version, and a TODO for migration:

```toml
[dependencies]
# EXPERIMENTAL: "experimental_enable_stop_iteration" enables HeadersBodyState
# for combined header+body access during response processing.
# Expected GA: PDK 1.8 (new state replaces this feature flag).
# TODO(pdk-1.8): Remove feature flag when GA lands; API may change.
pdk = { version = "1.7.0", features = ["experimental_enable_stop_iteration"] }
```

### Source Code

Every code block that uses experimental APIs must be annotated with a comment block:

```rust
// EXPERIMENTAL(experimental_enable_stop_iteration, GA: pdk-1.8):
// Uses ResponseHeadersBodyState to access both headers and body in response phase.
// When PDK 1.8 ships, replace with the stable API equivalent.
let headers_body_state = response_state.into_headers_body_state().await;
let handler = headers_body_state.handler();
let body = handler.body();
// ... analyze body ...
handler.set_header(":status", "403");
handler.set_body(rejection_body.as_bytes())?;
```

The annotation format is:

```
// EXPERIMENTAL(<feature_flag>, GA: <expected_ga_version>):
// <description of what the experimental API does>
// <migration note for when GA lands>
```

### Integration Tests

Tests exercising experimental behavior should also be annotated:

```rust
// EXPERIMENTAL(experimental_enable_stop_iteration, GA: pdk-1.8):
// This test validates response-phase status code override via HeadersBodyState.
// Update assertions if the GA API changes the state transition model.
#[pdk_test]
async fn response_moderation_returns_proper_status_code() -> anyhow::Result<()> {
    // ...
    assert_eq!(response.status(), 403, "HeadersBodyState allows setting status code on rejection");
    Ok(())
}
```

## Graduated Features (No Longer Experimental)

### `enable_stop_iteration` (was `experimental_enable_stop_iteration`)

**Status:** STABLE as of PDK 1.8. The `experimental_` prefix has been removed.

This feature provides `RequestHeadersBodyState` and `ResponseHeadersBodyState` that combine header and body access into a single state, allowing simultaneous read/write of both headers and body.

**PDK 1.8 stable enabling:**

```toml
pdk = { version = "1.8.0", features = ["enable_stop_iteration"] }
```

**Migration from 1.7 experimental:** Change the feature flag from `experimental_enable_stop_iteration` to `enable_stop_iteration` and update the PDK version. The API is the same -- use `into_headers_body_state()` to transition to the combined state.

For full documentation of this feature, see the **pdk-request-headers-bodies** skill (Approach 2: Stop Iteration).

## Available Experimental Features (PDK 1.9)

### `experimental`

**Expected GA:** Unknown

Enables body stream writing (`write_chunk` on `BodyStreamState`) and chunk construction. Less commonly needed for standard policies.

### `experimental_metrics`

**Expected GA:** Unknown

**What it provides:** The `pdk::metrics` module, backed by the `pdk-metrics-lib` crate. It exposes **counters** and **gauges** that integrate with the Omni Gateway's host metrics system, allowing custom policies to emit numeric metrics visible in whatever observability platform is connected to the gateway (e.g., Prometheus, Anypoint Monitoring).

**Enabling:**

```toml
pdk = { version = "1.9.0", features = ["experimental_metrics"] }
```

**Core types and traits:**

| Type / Trait | Description |
|---|---|
| `MetricsBuilder` | Injectable builder for creating counters and gauges. Injected into the `#[entrypoint]` configure function. |
| `MetricsInstanceBuilder` | Fluent builder for configuring a single metric with tags before calling `.build()` |
| `MetricsInstance` | A defined metric instance returned by `.build()` |
| `Metric` (trait) | Operations on a metric: `increase(i64)`, `set(u64)`, `get() -> u64` |
| `ReadinessInstance` | Convenience gauge that signals async policy initialization completed |
| `Readiness` (trait) | Provides `ready()` to set the readiness gauge to 1 |

**Metric types:**

- **Counter** — monotonically increasing value. Use `increase(offset)` to increment. Suitable for totals: requests processed, rejections issued, cache hits.
- **Gauge** — point-in-time value that can go up or down. Use `set(value)` to set an absolute value. Suitable for current state: API latency, cache size, queue depth.

Both types also support `get() -> u64` to read the current value.

**Automatic tagging:** Every metric automatically receives two default tags:
- `source=custom-metrics` — identifies the metric as originating from a custom policy
- `category=<filter_name>` — the policy's filter name from its metadata

Custom tags can be added via `.tag("key", "value")`. Default tags can be removed with `.skip_default_tags()`.

**Name/tag sanitization:** Dots (`.`), commas (`,`), and equals signs (`=`) in metric names and tag keys/values are replaced with underscores (`_`).

**Usage example:**

```rust
use pdk::metrics::{MetricsBuilder, Metric};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    metrics_builder: MetricsBuilder,
) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes)?;

    // Define metrics during configuration
    let request_rejections = metrics_builder
        .counter("content_safety_request_rejections")
        .tag("policy", "azure-content-safety")
        .build();

    let response_rejections = metrics_builder
        .counter("content_safety_response_rejections")
        .tag("policy", "azure-content-safety")
        .build();

    let api_latency = metrics_builder
        .gauge("content_safety_api_latency_ms")
        .build();

    let cache_hits = metrics_builder
        .counter("content_safety_cache_hits")
        .build();

    let filter = on_request(|request_state| {
        // In request filter: increment counter on rejection
        // request_rejections.increase(1);
        // api_latency.set(duration_ms);
    })
    .on_response(|response_state, request_data| {
        // In response filter: track response rejections
        // response_rejections.increase(1);
    });

    launcher.launch(filter).await?;
    Ok(())
}
```

**Readiness gauge example:**

```rust
use pdk::metrics::readiness::{ReadinessInstance, Readiness};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    readiness: ReadinessInstance,
) -> Result<()> {
    // ... async initialization (e.g., fetch remote config) ...

    // Signal that the policy is ready to handle traffic.
    // Sets gauge "<namespace>/<policy_name>/ready" to 1.
    readiness.ready();

    // ... launch filters ...
}
```

**Metric name format:** The final metric name registered with the host is formatted as `<name>,<tag1_key>=<tag1_value>,<tag2_key>=<tag2_value>,...` with tags sorted alphabetically by key.

**Limitations:**
- Metrics are registered with the Omni Gateway host — the exact integration path with external monitoring systems is not documented and may change
- No histogram type — only counters and gauges are available
- Metric instances must be created during configuration and shared into filter closures via move/clone semantics

## Migration Checklist (When GA Lands)

When an experimental feature graduates to GA:

1. **Search for all `EXPERIMENTAL(<feature_flag>` annotations** in the codebase
2. **Remove the feature flag** from `Cargo.toml`
3. **Update imports** if type/trait names changed at GA
4. **Update API calls** if method signatures changed
5. **Update integration tests** to remove experimental annotations
6. **Update documentation** (IMPLEMENTATION-GUIDE.md, ARCHITECTURE.md) to remove experimental caveats
7. **Run full test suite** to validate the migration
