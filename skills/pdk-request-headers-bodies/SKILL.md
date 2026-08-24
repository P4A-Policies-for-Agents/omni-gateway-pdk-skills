---
name: pdk-request-headers-bodies
description: Use when reading or writing request/response headers and bodies in PDK policies, covering HeadersHandler, BodyHandler, streaming, configured Omni/Envoy buffer limits, safe rewrite growth in PDK 1.10, WebSocket frames, and stop iteration via into_headers_body_state — including response-filter hangs with combined state.
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

**Default event-flow limit:** `into_body_state()` is documented for payloads up to 1 MB. Use the
streaming state for larger input; changing the Omni connection buffer is not documented as raising
this ordinary event-flow read limit.

PDK version and gateway compatibility matter when `set_body` reaches or exceeds the old 1 MB check:

- PDK before 1.10 normally rejects a replacement larger than 1 MB. Some experimental paths bypass
  that check and attempt the write; an oversized response can panic and an oversized request can
  produce a 413.
- PDK 1.10+ running on an Omni Gateway with the corresponding fix reads the configured
  `FLEX_DOWNSTREAM_CONNECTION_BUFFER_LIMIT_BYTES` limit when validating replacement writes and
  returns an error for a write at or above that limit instead of taking the panic path. Replacement
  bodies must be strictly smaller than the configured limit.
- Raising the connection buffer increases the finite supported size; it does not make rewrites
  unbounded. A policy cannot write more than the physical buffer or generate arbitrary extra body
  events.

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
- `BodyError::ExceededBodySize`: New body is at or above the maximum buffer size

### Streaming Bodies

For bodies too large to buffer, use the streaming body state **only when the operation can process
each chunk independently**. Streaming is read-only by default (cannot write) and does not affect
reading/writing headers. Per-chunk writing (`write_chunk` on the stream body state) exists only
behind the `experimental` Cargo feature — see the `pdk-experimental-feature` skill; without that
flag the stream is strictly read-only. If a decision needs bytes from a future chunk, retained data can still grow
until it hits a buffer limit. Whole-document transformations that produce a replacement body need a
declared maximum size, a chunk-compatible design, or an upstream service that performs the
transformation; PDK 1.10 fails oversized writes more safely but does not remove this constraint.

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

Use the `enable_stop_iteration` feature to simultaneously read and modify headers and body content.
Omni Gateway buffers the complete body before policy code runs. The default connection-buffer limit
is 1 MB; configure `FLEX_DOWNSTREAM_CONNECTION_BUFFER_LIMIT_BYTES` for a larger bounded stop-
iteration input or rewrite output. Exceeding the configured limit terminates the request before the
filter can complete.

### Enable Stop Iteration in Cargo.toml

```toml
[dependencies]
pdk = { version = "1.10.0", features = ["enable_stop_iteration"] }
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

### Caveat: the combined state can hang the response leg

`into_headers_body_state()` is convenient for a `response_filter` that must decide
what to write to the body based on a header (or vice versa) — but on some runtime
versions the combined headers+body state **hangs on the response leg**, so every
response-transforming request eventually returns an Envoy **504** (gateway timeout)
instead of the transformed response. This was observed on **Flex/Omni Gateway
1.12.1** and reproduced across multiple response-shaping policies; the request leg
was not affected. Treat it as version-dependent, not universal — but assume it until
you have verified the combined state works on the response leg for your target
runtime.

**Workaround — use the split event-flow states on the response path.** Do the header
work first, transition to the body state, then read/write the body:

```rust
async fn response_filter(response_state: ResponseState, /* … */) {
    // --- HEADERS PHASE ---
    let headers_state = response_state.into_headers_state().await;
    // Remove content-length so the gateway recomputes it from the new body.
    // Remove content-encoding too if you may replace a compressed body with plain
    // bytes, so the replacement is not mislabeled as gzip/br.
    headers_state.handler().remove_header("content-length");
    headers_state.handler().remove_header("content-encoding");

    // --- BODY PHASE --- (headers are frozen once you cross into the body state)
    let body_state = headers_state.into_body_state().await;
    let body = body_state.handler().body();
    // … transform `body` …
    let _ = body_state.handler().set_body(&new_body);
}
```

**Consequence — a body-content-dependent status rewrite is not possible on the split
flow.** `:status` is committed in the headers phase, before the body is read, so a
filter that only learns the body is bad *after* reading it can rewrite the **body**
(e.g. to a fail-closed error envelope) but must leave the status as the upstream sent
it. If you need a late status change you would need the combined state — which is the
one that hangs. Design response filters to be body-only on the affected runtimes.

---

## Documentation Reference

- Source (index): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers
- Source (event flow): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers-event
- Source (stop iteration): https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-headers-stop
- Examples: Stream Payload Policy, Stop Iteration Example Policy
- Release and payload-limit context: https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **Files:**
  - `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-headers.adoc`
  - `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-headers-event.adoc`
  - `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-headers-stop.adoc`
- **Snapshot:** 2026-08-24
