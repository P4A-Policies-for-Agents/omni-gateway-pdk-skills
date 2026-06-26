---
name: pdk-cors
description: Use when implementing Cross-Origin Resource Sharing (CORS) validation policies using the pdk::cors module to configure origin groups, allowed methods, headers, exposed headers, access control max age, and support credentials with W3C-compliant preflight and main request handling.
---

# Skill: Using CORS Library Functions

## Topic: Implementation

This skill covers how to use the PDK CORS library to validate incoming requests and protect resources using Cross-Origin Resource Sharing.

## Overview

The PDK CORS library complies with the CORS W3C recommendation standards. It is available from the `pdk::cors` module.

## Define a CORS Configuration

```rust
use pdk::cors;

fn create_cors_configuration() -> cors::Configuration {
    cors::Configuration::builder()
        .public_resource(false)
        .support_credentials(true)
        .origin_groups(vec![cors::OriginGroup::builder()
            .origin_group_name("default group".to_string())
            .plain_origins(vec!["http://www.the-origin-of-time.com".to_string()])
            .access_control_max_age(60)
            .headers(vec!["x-allow-origin".to_string()])
            .exposed_headers(vec!["x-forwarded-for".to_string()])
            .allowed_methods(vec![
                cors::AllowedMethod::builder()
                    .method_name("POST".to_string())
                    .allowed(true)
                    .build()
                    .unwrap(),
            ])
            .build()
            .unwrap()])
        .build()
        .unwrap()
}
```

### cors::Configuration Parameters

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
| `origin_groups` | Optional | Empty Vec | Groups of origins configurable with `cors::OriginGroup` |
| `public_resource` | Optional | `false` | Whether CORS config is applied as public resource |
| `support_credentials` | Optional | `false` | Whether CORS config supports credentials |

### cors::OriginGroup Parameters

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
| `plain_origins` | Optional | Empty Vec | Origins included in the group |
| `access_control_max_age` | Optional | 30 | Cache duration (seconds) for preflight response |
| `allowed_methods` | Optional | Empty Vec | Allowed HTTP methods via `cors::AllowedMethod` |
| `headers` | Optional | Empty Vec | HTTP headers for preflight requests |
| `exposed_headers` | Optional | Empty Vec | Headers browser JavaScript can access |

### cors::AllowedMethod Parameters

| Property | Required | Description |
|----------|----------|-------------|
| `method_name` | Required | Method name (CONNECT, DELETE, GET, OPTIONS, PATCH, POST, PUT, TRACE) |
| `is_allowed` | Required | Whether method is allowed |

## CORS Validation

Use `cors::Cors::check_headers()` to validate requests. Returns `Result<cors::Check, cors::CorsError>`.

`cors::Check` methods:
- `response_type()` → `cors::ResponseType` (`Preflight` or `Main`)
- `headers()` → headers to add to response
- `into_headers()` → consumes struct, returns headers

## Write a CORS Validation Policy

```rust
use pdk::*;
use pdk::cors;

#[entrypoint]
async fn configure(launcher: Launcher) -> Result<(), LaunchError> {
    let cors_configuration = create_cors_configuration();
    let filter = on_request(|rs| request_filter(rs, &cors_configuration))
        .on_response(response_filter);
    launcher.launch(filter).await
}

async fn request_filter(
    request_headers_state: RequestHeadersState,
    cors_configuration: &cors::Configuration<'_>,
) -> Flow<Vec<(String, String)>> {
    let cors = cors::Cors::new(cors_configuration);
    let request_headers = request_headers_state.handler().headers();
    let cors_result = cors.check(&request_headers);

    match cors_result {
        Ok(cors_check) => match cors_check.response_type() {
            cors::ResponseType::Preflight => {
                Flow::Break(Response::new(200).with_headers(check.into_headers()))
            }
            cors::ResponseType::Main => Flow::Continue(check.into_headers()),
        },
        Err(_) => Flow::Break(Response::new(200)),
    }
}

async fn response_filter(state: ResponseHeadersState, data: RequestData<Vec<(String, String)>>) {
    let RequestData::Continue(headers_to_add) = data else { return; };
    for (name, value) in headers_to_add.iter() {
        state.handler().set_header(name, value);
    }
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-cors
- Example: CORS Validation Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-cors.adoc`
- **Snapshot:** 2026-05-14
