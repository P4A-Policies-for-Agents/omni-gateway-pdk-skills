---
name: pdk-coding-best-practices
description: Use when implementing PDK policies with project-specific patterns for error handling, RequestData passing between filters, SSE streaming with BytesMut, body modification with Content-Length handling, JSON-RPC envelopes, telemetry with logger and MetricsBuilder, PolicyViolations reporting, StreamProperties for inter-policy communication, and test organization.
---

# Skill: PDK Coding Best Practices

## Topic: Implementation

This skill defines project-specific coding patterns and best practices for writing PDK policy code.

For pure code formatting and naming conventions, see the **pdk-code-style** skill.

## Policy File Structure

A typical policy `lib.rs` follows this layout:

```
lib.rs layout:
  1. Copyright header
  2. Module-level doc comments (//!)
  3. mod declarations
  4. External imports
  5. Workspace/PDK imports
  6. Local imports
  7. Constants
  8. Filter functions (request_filter, response_filter)
  9. Helper functions
  10. #[entrypoint] configure function
  11. #[cfg(test)] mod tests
```

Use `//!` module-level doc comments at the top of `lib.rs` (after copyright) to describe the policy's purpose:

```rust
// Copyright 2025 Salesforce, Inc. All rights reserved.

//! Anthropic transcoding policy.
//!
//! Transforms requests/responses between the unified LLM Gateway
//! format and the Anthropic Messages API format.

mod generated;
```

Remove boilerplate template comments left by code generation (e.g., `// This filter shows how to log a specific request header.`). These should not appear in production policy code.

## Error Handling

- Use `anyhow::Result` for the entrypoint return type
- `unwrap()` is acceptable for infallible parse operations (e.g., known-valid string constants parsed as `EntityTypeName`)
- For user-facing errors, use `Flow::Break(Response::new(status_code))` with appropriate HTTP status
- Log errors with `logger::error!()` before returning them
- Match on `Result` explicitly rather than using `?` when you need to handle the error case differently

## Request-to-Response Data Passing

Use the `RequestData<T>` enum to carry data from `request_filter` to `response_filter`. This is the standard pattern when the response filter needs context from the request phase:

```rust
enum RequestData {
    Continue(RequestContext),  // Request passed — carry context forward
    Break,                     // Request was short-circuited by this policy
    Cancel,                    // Request was cancelled
}

async fn request_filter(state: RequestState) -> Flow<RequestData> {
    // ... process request ...
    Flow::Continue(RequestData::Continue(context))
}

async fn response_filter(state: ResponseState, request_data: RequestData) {
    let RequestData::Continue(context) = request_data else {
        return Flow::Continue(());
    };
    // ... process response using context ...
}
```

Wire both filters in the entrypoint:

```rust
let filter = on_request(|rs| request_filter(rs, &config))
    .on_response(|rs, rd| response_filter(rs, rd));
```

## SSE Streaming

Many policies process Server-Sent Events (SSE) in response bodies. Use the `BufReader`/`BytesMut` chunk-by-chunk pattern:

```rust
use bytes::BytesMut;
use tokio::io::BufReader;

async fn response_filter(state: ResponseState, request_data: RequestData) -> Flow<()> {
    // ...
    let body_state = header_state.into_body_state().await;
    let mut remaining = BytesMut::new();

    loop {
        let chunk = body_state.read_chunk().await;
        match chunk {
            Some(bytes) => {
                remaining.extend_from_slice(&bytes);
                // Process complete SSE events from remaining buffer
                while let Some(event) = extract_next_event(&mut remaining) {
                    process_event(event);
                }
            }
            None => break,
        }
    }
    // ...
}
```

When modifying SSE events, ensure Content-Type remains `text/event-stream` and each event ends with `\n\n`.

## Body Modification & Content-Length

When a policy modifies the request or response body, the original `Content-Length` header becomes stale. **Always remove it before writing the new body**, so the gateway falls back to chunked transfer encoding:

```rust
const CONTENT_LENGTH_HEADER: &str = "content-length";

// Remove stale Content-Length before writing modified body
handler.remove_header(CONTENT_LENGTH_HEADER);
if let Err(e) = handler.set_body(new_body.as_bytes()) {
    logger::error!("Failed to set body: {:?}", e);
}
```

Define `CONTENT_LENGTH_HEADER` as a module-level constant — do not sprinkle the literal string `"content-length"` through the code.

When constructing a **new response** (e.g., short-circuiting with `Flow::Break`), set `Content-Length` to the actual body size:

```rust
const CONTENT_TYPE_HEADER: &str = "content-type";
const CONTENT_LENGTH_HEADER: &str = "content-length";
const APPLICATION_JSON: &str = "application/json";

Response::new(200)
    .with_headers(vec![
        (CONTENT_TYPE_HEADER.to_string(), APPLICATION_JSON.to_string()),
        (CONTENT_LENGTH_HEADER.to_string(), body.len().to_string()),
    ])
    .with_body(body)
```

## JSON-RPC Patterns

MCP and A2A policies use JSON-RPC 2.0 for request/response handling. Define minimal envelope types in the policy crate (or in a small shared module within it):

```rust
// In a `jsonrpc.rs` module local to the policy:
use serde::{Deserialize, Serialize};
use serde_json::value::RawValue;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JsonRpcRequest<'a> {
    pub method: &'a str,
    pub params: Option<&'a RawValue>,
    pub id: Option<&'a RawValue>,
    pub jsonrpc: Option<&'a str>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JsonRpcResponse {
    pub jsonrpc: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub id: Option<JsonRpcId>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<RpcError>,
}

#[derive(Clone, Debug, Deserialize, Serialize)]
pub struct RpcError {
    pub code: i32,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<serde_json::Value>,
}
```

```rust
use crate::jsonrpc::{JsonRpcRequest, JsonRpcResponse, RpcError};

// Parse incoming request
let json_request: JsonRpcRequest = from_slice(body_bytes.as_slice())?;

// Build error response
let error = RpcError {
    code: -32000,
    message: "Error description".to_string(),
    data: None,
};
let response = JsonRpcResponse::error(response_id, error);

// Serialize response
let body = serde_json::to_string(&response).unwrap_or_else(|_| {
    r#"{"jsonrpc":"2.0","error":{"code":-32603,"message":"Internal error"}}"#.to_string()
});
```

Standard JSON-RPC error codes: `-32000` for application errors, `-32603` for internal errors.

## Telemetry & Metrics

### Elapsed Time Logging

LLM Gateway policies measure execution time with `Instant::now()` and log it through `pdk::logger`:

```rust
use std::time::Instant;
use pdk::logger;

async fn request_filter(state: RequestState) -> Flow<RequestData> {
    let start = Instant::now();
    // ... policy logic ...
    logger::info!("[accessLog] {} elapsed={:?}", POLICY_NAME, start.elapsed());
    Flow::Continue(RequestData::Continue(context))
}
```

### Policy Name Constant

Define a `POLICY_NAME` constant for consistent logging and telemetry:

```rust
const POLICY_NAME: &str = "my-policy-name";
```

### Shared Metrics Builder

Use `RefCell<MetricsBuilder>` when metrics need to be accumulated across request and response filters:

```rust
use std::cell::RefCell;

let metrics_builder = RefCell::new(MetricsBuilder::new());
let filter = on_request(|rs| request_filter(rs, &metrics_builder))
    .on_response(|rs, rd| response_filter(rs, rd, &metrics_builder));
```

## Policy Violations

Use `PolicyViolations` from the PDK to report rule breaches without stopping execution:

```rust
use pdk::hl::PolicyViolations;

fn report_violation(handler: &dyn HeadersHandler, message: &str) {
    let violations = PolicyViolations::new();
    violations.add_violation(message);
    handler.set_policy_violations(violations);
}
```

This allows upstream policies or the gateway to collect and act on violations while the request continues.

## Stream Properties

Use `StreamProperties` for passing data between policies in the filter chain. One policy writes properties that another reads:

```rust
// Writing (e.g., in schema-validation policy)
stream_properties.write_property(
    &["mcp", "request", "method"],
    method_name.as_bytes(),
);

// Reading (e.g., in transcoding-router policy)
fn read_property_as_string(
    stream_properties: &StreamProperties,
    path: &[&str],
) -> Option<String> {
    stream_properties
        .read_property(path)
        .map(|bytes| String::from_utf8_lossy(bytes.as_slice()).to_string())
}
```

Property paths are arrays of string segments (e.g., `&["mcp", "request", "method"]`). Use descriptive, namespaced paths to avoid collisions between policies.

## Tests

### Test Module Organization

Place tests in `#[cfg(test)] mod tests` blocks at the bottom of each file. Multiple test modules per file are allowed for logical grouping:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_basic_functionality() { ... }
}

#[cfg(test)]
mod config_tests {
    use super::*;

    #[test]
    fn test_valid_config() { ... }
}

#[cfg(test)]
mod error_tests {
    use super::*;

    #[test]
    fn test_invalid_input() { ... }
}
```

### Test Naming

- Prefix all test functions with `test_`
- Do **not** use `should_` prefix (e.g., `should_parse_model`) — this exists in some older policies but is not the project convention
- Use descriptive snake_case names: `test_is_authorized_denies_invalid_access`
- Group related assertions in a single test when they share setup

### Test Helpers

Define test helper functions (non-`#[test]`) within the test module:

```rust
#[cfg(test)]
mod tests {
    fn create_test_principal() -> Entity { ... }
    fn create_test_context() -> Context { ... }

    #[test]
    fn test_authorization() {
        let principal = create_test_principal();
        ...
    }
}
```

## Common Pitfalls

Scaffolded projects (PDK 1.9.0+) ship an `AGENTS.md` documenting proxy-wasm runtime rules and recurring policy bugs. Keep these in mind alongside the patterns above — they are not enforced by the compiler:

- **State machine consumes ownership.** `RequestState` → `RequestHeadersState` → `RequestBodyState` (and the response-side equivalents) each transition consumes the previous state. Read everything you need from headers before transitioning to the body — you cannot go back.
- **Check `contains_body()` before reading or writing the body.** On a bodyless request (GET, HEAD, empty POST) `.body()` returns an empty buffer and writes will not reach upstream. This applies to the SSE and body-modification patterns above — guard the body transition.
- **Header names are case-insensitive.** Lowercase both sides before comparing (`name.to_ascii_lowercase()`). This is why the `CONTENT_LENGTH_HEADER` / `CONTENT_TYPE_HEADER` constants above are defined lowercase.
- **`Flow::Break(response)` rejects, `Flow::Continue(())` allows.** Inverting these is a security hole — an auth filter that returns `Continue` on failure passes the unauthenticated request upstream.
- **Decide explicitly how `HttpClient` errors are handled** (timeout, DNS failure, upstream 5xx). Fail-open vs fail-closed is a security decision — surface it in the policy config, do not silently swallow `.await` errors.

## Documentation Reference

- Enhanced: 2026-03-28 — extracted from `rust-code-style` skill and enriched with patterns from `pdk-custom-policy-examples`
- Updated: 2026-06-23 — added Common Pitfalls section (PDK 1.9.0 scaffolded `AGENTS.md`)
