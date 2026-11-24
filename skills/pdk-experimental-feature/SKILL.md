---
name: pdk-experimental-feature
description: Use when working with PDK experimental Cargo features, including the PDK 1.11 experimental aggregate, body-slice access (experimental_body_slice_access), data-store encryption (experimental_datastorage_encryption), contracts v2, data-storage formats and sync, body-limit-check bypass, metrics cadence/events/spans, WebSockets, mandatory code annotations, and migration checklists for GA.
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

## Available Experimental Features (PDK 1.11)

PDK 1.11 exposes these experimental feature flags (verified against the `pdk` 1.11.0 `[features]` table):

| Feature | Enables |
|---|---|
| `experimental` | Aggregate: core/classy experimental APIs, contracts v2, data-storage formats and encryption, body-slice access, metrics, and WebSockets |
| `experimental_body_slice_access` | Read or write a slice of an HTTP body (`body_size`, `body_range`, `set_body_range`). New in 1.11 |
| `experimental_contracts_v2` | Contracts v2 library surface |
| `experimental_datastorage_encryption` | Ask Flex to encrypt a remote data store at rest (`RemoteDataStorage::encrypted`). New in 1.11 |
| `experimental_datastorage_formats` | Experimental data-storage formats |
| `experimental_disable_body_limit_check` | Bypasses the PDK body-size guard; does not remove the Omni/Envoy physical buffer limit |
| `experimental_metrics` | Custom policy counters, gauges, readiness metrics, cadence, OTLP events, and tracing spans |
| `experimental_storage_sync` | Experimental synchronous data-storage support |
| `experimental_websocket` | Open-beta WebSocket filters and frame APIs |

Enable the narrowest feature that provides the API you need. The aggregate `experimental` feature
is substantially broader than body-stream writing and can increase API/behavior exposure. In 1.11 it
also turns on body-slice access and store encryption, so a policy that enables it for metrics also
gets both.

### `experimental`

**Expected GA:** Unknown

Enables the aggregate set shown above, including WebSockets, metrics, contracts v2, experimental
data-storage formats and encryption, body-slice access, and core/classy experimental APIs. Do not use it
when a narrower flag works.

### `experimental_body_slice_access`

**Expected GA:** Unknown

Adds `body_size`, `body_range` and `set_body_range` to `BodyHandler`. Writes are only possible at the
start or end of the body, and `set_body_range` does not update `content-length`. The full rules and an
example are in [[pdk-request-headers-bodies]] (Experimental: Read or Write a Body Slice). Enable it
only when you need to peek at or append to a body without a full rewrite.

### `experimental_datastorage_encryption`

**Expected GA:** Unknown

Lets a remote data store request at-rest encryption (`store_builder.remote(...).encrypted(true)`).
Flex 1.15+ on Redis, filesystem or distributed shared storage seals the values; other backends and older
gateways store plaintext without reporting it. Every handle on the store must use the same flag, and
turning encryption on for an existing store makes its old values unreadable (`Undecryptable`). See
[[pdk-data-storage]] (Experimental: Encrypt a Remote Store at Rest).

### `experimental_disable_body_limit_check`

**Expected GA:** Unknown

This flag only removes the PDK body-size guard. It does **not** remove the Omni/Envoy physical buffer
limit, and no experimental flag makes rewrite size unbounded. The real-world case that reaches for it
is a **large-body transcoding policy** — the latest MCP transcoding is the cited example — which
enables the bypass and then writes the converted body.

Failure modes when a write exceeds the buffer:

- **PDK < 1.10 with the bypass:** the 1 MB check is skipped and the write is attempted. An oversized
  **response** body **panics**; an oversized **request** body makes Envoy return a **413**.
- **PDK 1.10 on an Omni with the matching fix:** it reads the configured buffer size and **fails the
  oversized write gracefully** (no panic) — the write still fails past the limit.

The correct fix for a legitimately larger payload is not this bypass but raising the Omni buffer,
`FLEX_DOWNSTREAM_CONNECTION_BUFFER_LIMIT_BYTES` (managed Flex Gateway UI: **"Global connection buffer
limit"**), to the maximum converted size. That limit is per connection with no hard ceiling other
than host memory (10 MB is a valid value). See [[pdk-request-headers-bodies]].

### `experimental_metrics`

**Expected GA:** Unknown

Note: `pdk::metrics` is only exposed when `experimental_metrics` is enabled (see `pdk/src/lib.rs` in the
1.11.0 crate).

**What it provides:** The `pdk::metrics` module, backed by the `pdk-metrics-lib` crate. It exposes **counters** and **gauges** that integrate with the Omni Gateway's host metrics system, allowing custom policies to emit numeric metrics visible in whatever observability platform is connected to the gateway (e.g., Prometheus, Anypoint Monitoring).

**Enabling:**

```toml
pdk = { version = "1.11.0", features = ["experimental_metrics"] }
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

**Cadence, events and spans (PDK 1.11+):**

- `.cadence(MetricCadence::High | Standard | Extended)` on a `MetricsInstanceBuilder` sets how often the
  value is reported. The default is `Standard`.
- `pdk::metrics::event` (`Event::new(name)` with `.severity`, `.body`, `.attribute`, `.trace_id`) builds an
  OTLP log record, and `HttpEventExporter` (`.api_name`, `.timeout`) sends it. This is an experimental
  export path with no GA contract.
- `pdk::metrics::span` provides `DownstreamSpan` and `UpstreamSpan` for tracing tags: `set_tag`, `set_name`,
  and `adopt_decorator_header` (downstream) or `clear_tags` (upstream).

Check the exact signatures in the `pdk-metrics-lib` 1.11.0 source before relying on them; the docs page
did not describe these APIs at snapshot time.

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
