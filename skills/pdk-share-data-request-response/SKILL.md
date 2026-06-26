---
name: pdk-share-data-request-response
description: Use when passing data from the request function to the response function within the same policy using Flow::Continue(data) in the request filter and RequestData enum (Continue, Break, Cancel) in the response filter.
---

# Skill: Sharing Data Between Requests and Responses

## Topic: Implementation

This skill covers how to pass data between the request and response functions using PDK's `RequestData` enum.

## Overview

To share a status between request and response functions:
- The request function returns `Flow::Continue` with a data parameter
- The response function receives `RequestData` containing the final status and data

## RequestData Enum

`RequestData` has three possible values:

- `RequestData::Continue(data)`: Request function ended with `Flow::Continue(data)`
- `RequestData::Break`: Request function ended with `Flow::Break`
- `RequestData::Cancel`: Request function was cancelled by upstream policies

## Example

```rust
async fn request_filter(state: RequestState) -> Flow<String> {
    let state = state.into_headers_state().await;
    let handler = state.handler();
    if handler.header("authorization").is_some() {
        Flow::Continue(state.path())
    } else {
        Flow::Break(
            Response::new(401)
                .with_headers(vec![(
                    "WWW-Authenticate".to_string(),
                    "Bearer realm=\"oauth2\"".to_string(),
                )])
                .with_body(r#"{ "error": "token was not present"}"#),
        )
    }
}

async fn response_filter(state: ResponseState, request_data: RequestData<String>) {
    let RequestData::Continue(path) = request_data else {
        return;
    };

    let state = state.into_headers_state().await;
    logger::info!("Path: {path}, Status: {}", state.status_code());
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-share-data

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-share-data.adoc`
- **Snapshot:** 2026-05-14
