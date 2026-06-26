---
name: pdk-request-headers-bodies
description: Use when reading or writing request/response headers and bodies in PDK policies, covering the event flow approach (HeadersHandler, BodyHandler, streaming bodies for >1MB payloads, WebSocket frame bodies) and the stop iteration approach (enable_stop_iteration feature for simultaneous header-body access via into_headers_body_state).
---

# Skill: Reading and Writing Request Headers and Bodies

## Topic: Implementation

This skill covers how to read and write request/response headers and bodies in a PDK custom policy implementation.

PDK provides two separate approaches for reading and writing headers and bodies. Choose based on your use case:

- **Event Flow method** (default): Use when you don't need to read the request body and don't want to buffer the entire payload, or when you need to stream the request payload.
- **Stop Iteration method** (PDK 1.8+, requires feature flag): Use when your policy must read and write headers and bodies at the same time.

---

## Approach 1: Event Flow (Default)

### Event Flow

When filtering requests and responses, Proxy Wasm splits handling headers and body into two events in a specific order. For an API with two policies:

1. Policy 1 handles request header event
2. Policy 2 handles request header event
3. Backend receives request headers
4. Policy 1 handles request body event
5. Policy 2 handles request body event
6. Backend receives request body
7. Backend sends response
8. Policy 2 handles response header event
9. Policy 1 handles response header event
10. Client receives response headers
11. Policy 2 handles response body event
12. Policy 1 handles response body event
13. Client receives response body

**Limitations:**
- Policies can't modify headers after reading the body
- All policies must fully process the header event before reading the body
- Reject requests based on headers only to prevent data reaching the backend

In source code, `let body_state = headers_state.into_body_state().await;` separates header and body events.

### Read and Write Request Headers

Transform `RequestState` or `ResponseState` to header state via `into_headers_state()`. Then use `HeadersHandler` trait:

```rust
pub trait HeadersHandler {
    fn headers(&self) -> Vec<(String, String)>;
    fn header(&self, name: &str) -> Option<String>;
    fn add_header(&self, name: &str, value: &str);
    fn set_header(&self, name: &str, value: &str);
    fn set_headers(&self, headers: Vec<(&str, &str)>);
    fn remove_header(&self, name: &str);
}
```

Example:

```rust
async fn request_filter(request_state: RequestState, _config: &Config) {
    let headers_state = request_state.into_headers_state().await;
    let headers_handler = headers_state.handler();
    let old_value = headers_handler.header("request-header").unwrap_or_default();

    let new_value = "--replaced--";
    logger::info!("Old request header value: {old_value}, New value: {new_value}");
    headers_handler.set_header("request-header", new_value);
}
```

Envoy handles method, scheme, path, authority, and status as headers. Access via:

```rust
let method = headers_state.method();
let scheme = headers_state.scheme();
let authority = headers_state.authority();
let path = headers_state.path();
// For response:
let status = headers_state.status_code();
```

### Read and Write Request Bodies

**NOTE:** Limited to payloads of 1MB or smaller. For larger payloads, use streaming bodies.

Transform to body state via `into_body_state()`:

```rust
let body_state = request_state.into_body_state().await;
// Or from headers state:
let body_state = headers_state.into_body_state().await;
```

Use `BodyHandler` trait:

```rust
pub trait BodyHandler {
    fn body(&self) -> Vec<u8>;
    fn set_body(&self, body: &[u8]) -> Result<(), BodyError>;
}
```

**Important:** Cannot access headers and body at the same time. Read headers first, save values, then read body. Complete all header modifications before reading body. Remove `content-length` header before modifying body.

`BodyHandler::set_body()` may fail with:
- `BodyError::BodyNotSent`: No body in current HTTP Flow (e.g., GET request)
- `BodyError::ExceededBodySize`: New body exceeds maximum buffer size

### Streaming Bodies

For bodies larger than 1MB, use the streaming body state. Streaming is read-only (cannot write). Does not affect reading/writing headers.

```rust
let body_stream_state = request_state.into_body_stream_state().await;

let mut stream = body_stream_state.stream();
while let Some(chunk) = stream.next().await {
    let chunk_bytes = chunk.into_bytes();
    // ... process chunk_bytes ...
}

// Or collect all chunks:
let collect = stream.collect().await;
```

### WebSocket Frame Bodies

The approaches above apply to HTTP request/response bodies. WebSocket payloads are handled
differently: after the connection upgrades, body bytes arrive as a stream of WebSocket frames
on `UpstreamState` (client→server) and `DownstreamState` (server→client), not as a single
`BodyHandler` payload. Read raw bytes with `state.bytes()`, accumulate partial frames with
`state.accumulate().await`, decode/encode frames with `pdk::websockets::{Decoder, Encoder}`,
write with `state.set_body(&encoded)`, and advance with `state.next().await`. Enable this with
the `experimental_websocket` PDK feature. For full coverage see the **pdk-websockets** skill.

---

## Approach 2: Stop Iteration (PDK 1.8+)

Use the `enable_stop_iteration` feature to simultaneously read and modify headers and body content. This approach supports bodies up to 1 MB only.

### Enable Stop Iteration in Cargo.toml

```toml
[dependencies]
pdk = { version = "1.9.0", features = ["enable_stop_iteration"] }
```

### Read and Write Headers and Body Together

Use `into_headers_body_state()` to transition to a combined headers-body state. The handler implements both `HeadersHandler` and `BodyHandler` traits:

```rust
use pdk::hl::*;

async fn request_filter(request_state: RequestState) -> Flow<()> {
    let state = request_state.into_headers_body_state().await;

    // Access both headers and body through the unified handler
    state.handler().set_header("x-custom-header", "value");

    if state.contains_body() {
        let _ = state.handler().set_body("modified body".as_bytes());
    }

    Flow::Continue(())
}
```

### Handle Requests Without Body

```rust
async fn request_filter(request_state: RequestState) -> Flow<()> {
    let state = request_state.into_headers_body_state().await;

    if state.contains_body() {
        let body = state.handler().body();
        // Process body
    }

    // Attempting to set body on a request without body returns an error
    if let Err(BodyError::BodyNotSent) = state.handler().set_body("data".as_bytes()) {
        state.handler().set_header("x-data", "data");
    }

    Flow::Continue(())
}
```

---

## Documentation Reference

- Source (index): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers
- Source (event flow): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers-event
- Source (stop iteration): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers-stop
- Examples: Stream Payload Policy, Stop Iteration Example Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **Files:**
  - `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-headers.adoc`
  - `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-headers-event.adoc`
  - `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-headers-stop.adoc`
- **Snapshot:** 2026-05-14
