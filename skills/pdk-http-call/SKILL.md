---
name: pdk-http-call
description: Use when making HTTP calls from PDK custom policies to external services, including defining service parameters with format service in gcl.yaml, injecting HttpClient into entrypoint or wrapped functions, and performing requests with path, headers, body, and HTTP methods.
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
