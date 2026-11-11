---
name: pdk-http-call
description: Use when making HTTP calls from PDK custom policies to external services, including defining service parameters with format service in gcl.yaml, registering the service at Flex init so the request routes at runtime, injecting HttpClient into entrypoint or wrapped functions, performing requests with path, headers, body, and HTTP methods, caching an outbound OAuth/JWT-bearer access token with single-flight renewal and failure backoff, calling an external guardrail or moderation service with timeouts and fail-open/fail-closed handling, and configuring outbound TLS trust (the tls-outbound PolicyBinding with trustedCA or skipValidation) to call an HTTPS upstream signed by a private or self-signed CA.
---

# Skill: Performing an HTTP Call

## Topic: Implementation

This skill covers how to make HTTP calls from a PDK custom policy to interact with external services.

## Define the HTTP Service

Define the service as a parameter in your schema definition (`gcl.yaml`):

```yaml
properties:
  externalService:
    type: string
    format: service
  endpointPath:
    type: string
```

## Register the Service at Flex Init (required for real gateways)

Defining a `format: service` parameter is **not enough** to route a request. On a real
(non-local) Omni/Flex Gateway the runtime uses a split model: a `#[pdk::hl::entrypoint_flex]`
`init` function runs once per replica at startup and must **register** every service it will
call via `abi.service_create(...)`. The `#[entrypoint] configure` function and the request
filter run later and only *use* the already-registered cluster. Omit the registration and the
`client.request(&config.my_service)` call has no cluster to route to — it fails at runtime even
though the policy built and passed unit tests.

```rust
// Runs once at startup, before any request. Registers the outbound cluster.
#[pdk::hl::entrypoint_flex]
fn init(abi: &dyn pdk::flex_abi::api::FlexAbi) -> Result<(), anyhow::Error> {
    // Deserialize the SAME config type `configure` uses (see rule below).
    let config: Config = serde_json::from_slice(abi.get_configuration()).map_err(|err| {
        anyhow::anyhow!(
            "Failed to parse configuration '{}'. Cause: {}",
            String::from_utf8_lossy(abi.get_configuration()),
            err
        )
    })?;

    // Register only when a URL was actually provided; an empty authority means "unset".
    if !config.my_service.uri().authority().is_empty() {
        abi.service_create(config.my_service)?;
    }

    abi.setup()?; // keep this — it wires the standard filter setup the scaffold expects
    Ok(())
}
```

**The init ↔ configure contract.** `init` and `configure` MUST derive the `Service` from the
**same config type and the same field**. The cluster name is derived purely from the URL
**authority** (host\[:port]); if `init` registers a cluster from one URL shape and `configure`
builds its request `Service` from another (a different field, a stripped path, a dropped explicit
port), the runtime request targets a cluster name that was never registered → silent routing
failure. Practical rules:

- Deserialize the service with `#[serde(deserialize_with = "pdk::serde::deserialize_service")]`
  into a `pdk::hl::Service` field — do not re-parse a raw URL string differently in each entrypoint.
- **Preserve the full URI** — explicit port (even `:80`/`:443`) and path. The authority selects
  the cluster; the path is read back for the request and by libraries that inspect the full URL
  (e.g. IdP-endpoint detection). Stripping either breaks routing or downstream logic.
- `init` typically registers **without** semantic validation (it only needs the service);
  `configure` runs full `validate()` after deserialization.

This registration step is not covered by the public docs — it surfaces only at deploy time, so
treat it as mandatory whenever a policy makes an outbound call.

## Timer Interoperability (PDK 1.10+)

PDK 1.10 fixes context routing for HTTP calls triggered after awaiting a timer tick. Such calls are
now routed through the request context rather than being incorrectly delivered to the root context
handler. Use PDK 1.10 or later for WebSocket or request flows that await a tick and then call an
HTTP service.

## Make HTTP Requests

Access the defined service in the `Config` struct and perform requests via the HTTP client:

```rust
let response = client
    .request(&config.external_service)
    .path(&config.endpoint_path)
    .headers(vec![("Content-Type", "application/json")])
    .body(r#"{"key": "value"}"#.as_bytes())
    .put().await?;
```

## Inject the HTTP Client

**Option 1 — Inject into `#[entrypoint]`** (available in both request and response filters):

```rust
#[entrypoint]
async fn configure(launcher: Launcher, Configuration(bytes): Configuration, client: HttpClient) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes).unwrap();

    let filter = on_request(|request_state| request_filter(request_state, &config, &client))
        .on_response(|response_state, request_data| {
            response_filter(response_state, request_data, &config, &client)
        });

    launcher.launch(filter).await?;
    Ok(())
}
```

**Option 2 — Inject into wrapped functions** (available only in that filter):

```rust
async fn request_filter(state: RequestState, conf: &Config, client: HttpClient) {
    // ...
}

#[entrypoint]
async fn configure(launcher: Launcher, Configuration(bytes): Configuration) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes).unwrap();

    launcher
        .launch(on_request(|request, client| {
            request_filter(request, &config, client)
        }))
        .await?;
    Ok(())
}
```

## Caching an outbound access token (single-flight renewal)

A policy that has to obtain a token before calling a backend (client credentials, an RFC 7523
JWT-bearer assertion, a token exchange) must not mint one per request. Keep the token in a
data-storage entry ([[pdk-data-storage]]) and let at most one in-flight request renew it, guarded by
a `TryLock` (`pdk::lock`). Every other request keeps serving the cached token meanwhile.

```rust
const SERVE_MARGIN_SECS: u64 = 60;   // stop serving 60s before expiry
const RENEW_MARGIN_SECS: u64 = 120;  // start renewing 120s before expiry

// lock = lock_builder.new("token-renew".into())
//            .expiration(Duration::from_millis(call_timeout_ms + 1_000)) // outlive one mint call
//            .build();

match cached_entry {
    // Fresh: serve.
    Some(e) if now < e.renew_at => Ok(e.token),
    // Renewal window: one request renews, the rest keep serving.
    Some(e) if now < e.serve_until => match lock.try_lock() {
        None => Ok(e.token),
        Some(_guard) => match mint_and_store().await {
            Ok(t) => Ok(t),
            Err(_) if now_secs() < e.serve_until => Ok(e.token), // renewal failed, old token still valid
            Err(err) => Err(err),
        },
    },
    // Nothing usable: mint, unless a failure window is open.
    _ => mint_and_store().await,
}
```

- `serve_until = expiry − 60s`, `renew_at = expiry − 120s`, so renewal starts while the old token
  is still being served.
- **Remember failures too.** Store a failure entry with a close time so that, under load, a broken
  token endpoint isn't hit by every request. Use a fixed window (e.g. 30s) for non-retryable errors
  (bad credentials, 4xx) and exponential backoff (1s → 30s cap) for retryable ones (timeout,
  connection, 5xx). While the window is open, return the stored error without calling.
- Write with `StoreMode::Cas(version)` and retry on `CasMismatch`, so two replicas sharing remote
  storage don't clobber each other.
- **A bad signing key or credential must not fail `configure`.** Turn it into a per-request auth
  error (a 5xx or a 401 response from the filter), so the policy still loads and the misconfiguration
  is visible on every call instead of the API silently losing its policy.
- Record the auth-latency metric only on a real mint, not on cache hits.

## Calling an external guardrail / moderation service

A policy that sends the prompt or response to an external moderation or guardrail API (and blocks
on its verdict) needs an explicit failure contract:

| Concern | Do |
|---|---|
| Timeout | `.timeout(Duration::from_millis(config.timeout_ms))` on every call; never inherit an unbounded default |
| Failure classes | classify into a closed set (auth, bad request, not found, timeout, connection, upstream 5xx), and use that class, not the raw error, in logs, metrics and any signal header |
| `fail_open` config flag | `true`: continue unmoderated and carry the failure reason to the response filter (for a header or metric). `false`: reject with a 403 and a generic message. Make the flag explicit, defaulting to the safer choice for the use case |
| Error text | cap previews of the service's response body (e.g. 200 chars); never include the API key, bearer token, or the moderated content itself |
| SSE responses | response moderation needs the **complete** text, so the stream must be buffered and reassembled from the `data:` frames before calling. That removes incremental streaming for that route, so document it. Pass through (don't moderate an empty string) when no text can be recovered |

Register the guardrail service at init like any other outbound call (above).

## Calling an HTTPS upstream with a private/self-signed CA (outbound TLS trust)

When a policy calls an HTTPS upstream whose certificate is signed by a private or self-signed CA,
the request fails cert validation by default — and you **cannot** fix it by installing certs on the
OS running Flex. Instead, attach outbound TLS config to the *service* the policy calls, via the
built-in `tls-outbound` policy bound to a `Service`. This is gateway configuration, not Rust:

```yaml
# A Service pointing at the HTTPS upstream the policy will call.
apiVersion: gateway.mulesoft.com/v1alpha1
kind: Service
metadata:
  name: my-upstream
  namespace: example
spec:
  address: https://secure-backend:443
---
# Outbound TLS trust for that Service.
apiVersion: gateway.mulesoft.com/v1alpha1
kind: PolicyBinding
metadata:
  name: my-upstream-tls
spec:
  targetRef:
    kind: Service
    name: my-upstream
    namespace: example
  policyRef:
    name: tls-outbound
  config:
    skipValidation: false      # true disables validation entirely (dev only)
    trustedCA: |               # PEM of the CA to trust for this upstream
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

The policy code is unchanged — build a `Service` from the same `name`/`namespace` and call it with
`HttpClient` as usual. Prefer `trustedCA` (trust a specific CA) over `skipValidation: true` (trust
anything), which is only appropriate in the local playground. In `pdk-debug-local`/`pdk-test-locally`
the same `Service` + `PolicyBinding` pair goes in the playground config so local runs mirror the
gateway.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-http-request
- Example: Simple OAuth 2.0 Validation Policy
- Outbound TLS pattern: the `tls-calls` PDK example (`pdk-custom-policy-examples`)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-http-request.adoc`
- **Release notes:** https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.10.0)
- **Snapshot:** 2026-08-24
- **Note:** the token-caching and guardrail-call sections (2026-11-11) are field-derived from
  MuleSoft's own LLM/auth policies, not from the docs page. They use only public 1.10.0 APIs
  (`pdk::lock`, `pdk::data_storage`, `HttpClient`).
