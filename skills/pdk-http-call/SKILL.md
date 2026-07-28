---
name: pdk-http-call
description: Use when making HTTP calls from PDK custom policies to external services, including defining service parameters with format service in gcl.yaml, registering the service at Flex init so the request routes at runtime, injecting HttpClient into entrypoint or wrapped functions, and performing requests with path, headers, body, and HTTP methods.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-http-request
- Example: Simple OAuth 2.0 Validation Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-http-request.adoc`
- **Snapshot:** 2026-04-23
