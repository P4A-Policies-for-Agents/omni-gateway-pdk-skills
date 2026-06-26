---
name: pdk-unit-tests
description: Use when writing deterministic unit tests for PDK policies with pdk-unit framework (1.8+), covering request/response lifecycle, WebSocket frame testing (1.9+), TraceBackend upstream verification, mocking HTTP/gRPC upstreams, testing client ID enforcement, time-based behavior with Clock/Timer, dw2pel config serialization, local_mode resilience testing, and JSON-RPC error validation.
---

# Skill: Writing Unit Tests with pdk-unit

## Topic: Testing

This skill covers how to write unit tests for PDK custom policies using the `pdk-unit` framework (PDK 1.8+). Unit tests exercise the full request/response lifecycle without a running Envoy proxy, WebAssembly runtime, or external infrastructure. WebSocket frame testing and testing non-PDK proxy-wasm policies require PDK 1.9+.

Benefits:
- **Fast feedback**: Tests run as ordinary Rust unit tests, completing in milliseconds
- **Deterministic**: No network, no containers, no race conditions
- **Full lifecycle coverage**: Exercises request headers, request body, response headers, and response body phases
- **Code coverage**: Works with standard coverage tools such as `cargo-tarpaulin`
- **Debugging**: Tests run as native Rust code, so you can attach a debugger or profiler

## Enable pdk-unit

Add `pdk-unit` as a dev-dependency in the policy's `Cargo.toml`. **In this monorepo, use the workspace dependency — do NOT pin a version locally:**

```toml
[dev-dependencies]
pdk-unit = { workspace = true, features = ["experimental"] }
```

### MCP / JSON-RPC handler policies — `enable_stop_iteration` is mandatory

If the policy returns `Flow::Break(response)` from inside a JSON-RPC handler — whether you use a workspace dispatcher framework (handler-per-method) or a hand-rolled MCP/A2A request filter (see [[pdk-mcp]], [[pdk-a2a]]) — you MUST add `enable_stop_iteration` to BOTH `pdk` and `pdk-unit`:

```toml
[dependencies]
pdk      = { workspace = true, features = ["experimental", "enable_stop_iteration"] }

[dev-dependencies]
pdk-unit = { workspace = true, features = ["experimental", "enable_stop_iteration"] }
```

Without this, `Flow::Break(response)` returned by a handler can't actually short-circuit the filter chain: the filter awaits body state that never arrives, and pdk-unit tests hang past 60s and get `SIGKILL`ed. **Symptom in test output**: many tests showing `has been running for over 60 seconds` followed by `process didn't exit successfully: ... signal: 9, SIGKILL`. If you see that, check this feature first.

### Testing non-PDK proxy-wasm policies (PDK 1.9+)

`pdk-unit` can now test any policy built on [`proxy-wasm-rust-sdk`](https://github.com/proxy-wasm/proxy-wasm-rust-sdk) **without migrating it to PDK first** — useful for establishing a behavioral baseline before migration or catching regressions during one. Outside this monorepo's workspace the dependency is pinned explicitly:

```toml
[target.'cfg(target_arch = "wasm32")'.dependencies]
proxy-wasm = "0.2.5"

# host-side stub replaces proxy-wasm for native test builds; NOT for wasm32
[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
pdk-proxy-wasm-stub = "1.9.0"

[dev-dependencies]
pdk-unit = { version = "1.9.0", default-features = false, features = ["proxy-wasm-rust-sdk"] }
```

Import the right crate per target in the policy code, and build tests against the stub:

```rust
#[cfg(target_arch = "wasm32")]
use proxy_wasm as pw;
#[cfg(not(target_arch = "wasm32"))]
use pdk_proxy_wasm_stub as pw;
```

For non-PDK policies use `with_context` instead of `with_entrypoint` — it takes a closure that builds the policy's root context:

```rust
let mut tester = UnitTestBuilder::default()
    .with_backend(UnitHttpResponse::new(200))
    .with_context(|| Box::new(MyRootContext));
```

## Repo Conventions

1. **File location**. Place tests in `<policy>/implementation/src/tests.rs` and wire them into `lib.rs` with:
   ```rust
   #[cfg(test)]
   mod tests;
   ```

2. **Module structure inside `tests.rs`**. All tests MUST live inside a top-level `mod tests { … }` block. Group related tests into sub-modules by domain when the file grows beyond a handful of tests — sub-modules let you run a specific group with `cargo test --lib tests::group_name`:

   ```rust
   // src/tests.rs
   mod tests {
       use super::*;
       use pdk_unit::{UnitHttpRequest, UnitHttpResponse, UnitTestBuilder};
       use serde_json::json;

       fn default_config() -> String { /* shared helpers visible to all sub-modules */ }

       mod request_filter {
           use super::*;
           #[test] fn passes_through_non_json() { /* … */ }
           #[test] fn rewrites_body() { /* … */ }
       }

       mod response_filter {
           use super::*;
           #[test] fn propagates_upstream_error() { /* … */ }
       }

       // Small policies with few tests can skip sub-modules
       // and put #[test] fns directly inside `mod tests`.
   }
   ```

3. **Running tests**:
   ```bash
   # All pdk-unit tests for one policy
   cd <policy>/implementation && cargo test --lib tests

   # Workspace-wide
   make implementations-unit-test
   ```

## Basic Request and Response Test

Build a `UnitTest` with `UnitTestBuilder`, send a `UnitHttpRequest`, and assert on the returned `UnitHttpResponse`:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpMessage};

#[test]
fn test_policy_allows_get() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{"allowed_methods": ["GET"]}"#)
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::get().with_path("/api/resource"),
    );

    assert_eq!(response.status_code(), 200);
}
```

## View Backend Requests with TraceBackend

Use `TraceBackend` to record requests that reach the upstream:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpResponse, UnitHttpMessage, TraceBackend, Backend};
use std::rc::Rc;

#[test]
fn test_policy_adds_header() {
    let backend = Rc::new(TraceBackend::new(UnitHttpResponse::new(200)));

    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{}"#)
        .with_backend(Rc::clone(&backend))
        .with_entrypoint(crate::configure);

    tester.request(UnitHttpRequest::get().with_path("/api/resource"));

    let upstream_req = backend.next().unwrap();
    assert_eq!(upstream_req.header("x-added-by-policy"), Some("true"));
}
```

## Return Custom Upstream Responses

Use `UnitHttpResponse` or a closure implementing `Backend` to mock upstream responses:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpResponse, UnitHttpMessage};

#[test]
fn test_policy_handles_upstream_error() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{}"#)
        .with_backend(|_req| {
            UnitHttpResponse::new(503)
                .with_body(r#"{"error":"unavailable"}"#)
        })
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::get().with_path("/api/resource"),
    );

    assert_eq!(response.status_code(), 503);
}
```

## Mock an HTTP Upstream

Register a backend for a specific authority using `with_http_upstream_from_authority`:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpResponse, UnitHttpMessage};

#[test]
fn test_policy_calls_external_service() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{"auth_url": "https://auth.example.com"}"#)
        .with_http_upstream_from_authority("auth.example.com", |_req| {
            UnitHttpResponse::new(200)
                .with_body(r#"{"valid":true}"#)
                .into()
        })
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::get().with_path("/secure"),
    );

    assert_eq!(response.status_code(), 200);
}
```

## Mock a gRPC Upstream

Use the `protobuf_grpc_backend` macro to derive a `GrpcBackend` implementation:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpMessage, protobuf_grpc_backend};

#[derive(Default)]
pub struct MockAuthService;

#[protobuf_grpc_backend]
impl MockAuthService {
    #[grpc_method(service = "AuthService", method = "Verify")]
    fn verify(&self, _req: VerifyRequest) -> VerifyResponse {
        VerifyResponse { authorized: true, ..Default::default() }
    }
}

#[test]
fn test_grpc_policy() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{}"#)
        .with_grpc_upstream_from_authority("grpc.backend.com", MockAuthService::default())
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::post().with_path("/api/data"),
    );

    assert_eq!(response.status_code(), 200);
}
```

## Test WebSocket Policies (PDK 1.9+)

Policies built with the WebSocket FilterBuilder hooks (`on_create`, `on_upgrade_upstream`, `on_upgrade_downstream`, `on_done` — see [[pdk-websockets]]) are tested with `pdk-unit`'s WebSocket support. WebSocket tests differ from HTTP tests: they establish an upgrade connection and exercise bidirectional frame exchange instead of a single request/response.

1. Configure the backend as `UnitHttpResponse::upgrade()` to simulate a successful upgrade handshake.
2. Call `tester.upgrade(UnitHttpRequest::upgrade())` instead of `request`. It returns an `UpgradeConnection`.
3. Drive frames in both directions via `conn.client()` and `conn.server()`.

```rust
use pdk_unit::{UnitFrame, UnitFrameType, UnitHttpRequest, UnitHttpResponse, UnitTest, UnitTestBuilder};

fn tester() -> UnitTest {
    UnitTestBuilder::default()
        .with_config("{}")
        .with_backend(UnitHttpResponse::upgrade())
        .with_entrypoint(crate::configure)
}

#[test]
fn text_frames_are_prefixed_with_counter() {
    let mut t = tester();
    let conn = t.upgrade(UnitHttpRequest::upgrade()).unwrap();

    // client → server: send a text frame, read what the upstream receives
    conn.client().send_to_server(UnitFrame::text("hello", true));
    let frame = conn.server().next().unwrap();
    assert_eq!(String::from_utf8_lossy(frame.data()), "1:hello");
}

#[test]
fn control_frames_pass_through_unchanged() {
    let mut t = tester();
    let conn = t.upgrade(UnitHttpRequest::upgrade()).unwrap();

    conn.client().send_to_server(UnitFrame::ping());
    assert_eq!(conn.server().next().unwrap().frame_type(), UnitFrameType::Ping);

    // server → client direction
    conn.server().send_to_client(UnitFrame::pong());
    assert_eq!(conn.client().next().unwrap().frame_type(), UnitFrameType::Pong);
}
```

`UpgradeConnection` exposes:
- `response()` — the upgrade response, for asserting handshake headers/status.
- `client()` — send frames to the server (`send_to_server`) and read frames coming back (`next`).
- `server()` — send frames to the client (`send_to_client`) and read frames the upstream received (`next`).

Construct frames with `UnitFrame`. The text/binary constructors take a `fin` flag marking the final fragment of a message:
- `UnitFrame::text(payload, fin)`, `UnitFrame::binary(payload, fin)`
- `UnitFrame::ping()`, `UnitFrame::pong()`, `UnitFrame::close()`

Inspect received frames with `frame.frame_type()` (`UnitFrameType::{Text, Binary, Ping, Pong, Close}`) and `frame.data()`. `next()` returns `Option<UnitFrame>`.

### Frame chunking and `set_chunk_size`

As of PDK 1.9.0 the default chunk size is **30 KB** (was 3 bytes), aligning with Envoy and cutting per-event overhead. To deliberately force mid-payload fragmentation — e.g. to prove your `on_upgrade_*` hook accumulates partial frames before processing — set a small chunk size before upgrading:

```rust
#[test]
fn incomplete_frames_are_accumulated_before_processing() {
    let mut t = tester();
    t.set_chunk_size(2); // force TCP fragmentation
    let conn = t.upgrade(UnitHttpRequest::upgrade()).unwrap();

    conn.client().send_to_server(UnitFrame::text("hello world", true));
    let frame = conn.server().next().unwrap();
    assert_eq!(
        String::from_utf8_lossy(frame.data()),
        "1:hello world",
        "frame must not be split mid-payload",
    );
}
```

`set_chunk_size` also applies to HTTP body delivery; use it when a test must verify body-accumulation logic across fragments.

## Test Client ID Enforcement

Use `add_contract_data` to seed the built-in Anypoint Platform stub with API contract data:

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpMessage};
use std::time::Duration;

#[test]
fn test_valid_client_id() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{}"#)
        .with_entrypoint(crate::configure);

    tester.add_contract_data(
        "my-client-id",
        "My App",
        Some("my-client-secret"),
        None::<String>,
    );

    tester.sleep(Duration::from_secs(20)); // advance internal clock for periodic task

    let response = tester.request(
        UnitHttpRequest::get()
            .with_path("/api/resource")
            .with_header("client_id", "my-client-id")
            .with_header("client_secret", "my-client-secret"),
    );

    assert_eq!(response.status_code(), 200);
}
```

## Test Time-Based Behavior

Unit tests use an internal clock. Use `tick()` and `sleep()` to advance time without actually waiting. Use `Clock::now` or `Timer::now` for time in the policy (not `SystemTime::now`):

```rust
use pdk_unit::{UnitTestBuilder, UnitHttpRequest, UnitHttpMessage};
use std::time::Duration;

#[test]
fn test_rate_limit_resets_after_window() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{"requests_per_minute": 1}"#)
        .with_entrypoint(crate::configure);

    let req = || UnitHttpRequest::get().with_path("/api/resource");

    let first = tester.request(req());
    assert_eq!(first.status_code(), 200);

    let second = tester.request(req());
    assert_eq!(second.status_code(), 429);

    // Advance time past the rate-limit window
    tester.sleep(Duration::from_secs(60));

    let third = tester.request(req());
    assert_eq!(third.status_code(), 200);
}
```

**NOTE:** Using `SystemTime::now` creates non-deterministic tests.

## Repo-specific helpers (`pdk_tests_utils::helpers`)

These helpers live in `libs/pdk-tests-utils/` and address gaps the framework alone doesn't cover. Prefer them over rolling your own equivalents.

### `dw2pel` — DataWeave config fields

Many policies declare config fields with `format: dataweave` in `gcl.yaml` (e.g. tool URI params, header values, body templates). The generated config deserializer expects a compiled `pdk::script::Expression`, **not** a plain string. Passing `"vars.params.foo"` directly in `with_config(...)` JSON fails with:

```
invalid type: string "...", expected PEL Expression
```

Use `pdk_unit::dw2pel()` to convert a DataWeave expression into the PEL wire format the deserializer accepts:

```rust
use pdk_unit::dw2pel;
use serde_json::json;

let config = json!({
    "tools": [{
        "name": "get_user",
        "path": "/users/{userId}",
        "method": "GET",
        "uriParams": [
            { "key": "userId", "value": dw2pel("vars.params.userId") }
        ]
    }]
});
let mut tester = UnitTestBuilder::default()
    .with_config(&config.to_string())
    .with_entrypoint(crate::configure);
```

Allowed inputs inside `dw2pel`: `payload`, `attributes`, `vars`, `authentication`.
Allowed functions: `++`, `--`, `contains`, `splitBy`, `trim`, `lower`, `upper`, `sizeOf`, `uuid`, `isEmpty`, `substringBefore`/`After`/`BeforeLast`/`AfterLast`, `toBase64`, `fromBase64`.

### `serialize_auth` — injecting the authentication object

When a policy reads the authentication object from the request context, inject it via `with_property` rather than inventing setters:

```rust
use pdk_tests_utils::helpers::serialize_auth;

#[test]
fn test_authenticated_user_can_call_tool() {
    let mut tester = UnitTestBuilder::default()
        .with_config(r#"{ "rules": ["permit(principal, action, resource);"] }"#)
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::post()
            .with_path("/tools")
            .with_header("content-type", "application/json")
            .with_property(vec!["authentication"], serialize_auth("alice"))
            .with_body(r#"{"jsonrpc":"2.0","method":"tools/call","id":1}"#),
    );
    assert_eq!(response.status_code(), 200);
}
```

### `configure_asset` + `EXCHANGE_CLUSTER_NAME` — Exchange-fetching policies

For policies that pull an asset from Exchange (e.g. schema-validation, MCP spec fetch), seed the asset service via `metadata(configure_asset)` and mock the Exchange HTTP cluster. Follow with `tester.tick();` to advance the periodic asset-collection timer **before** the first `.request(...)`:

```rust
use pdk_tests_utils::helpers::{configure_asset, EXCHANGE_CLUSTER_NAME};

let mut tester = UnitTestBuilder::default()
    .metadata(configure_asset)
    .with_http_upstream(EXCHANGE_CLUSTER_NAME, |_req| {
        UnitHttpResponse::new(200).with_body(MCP_SPEC).into()
    })
    .with_config(r#"{"validateToolSchema": true}"#)
    .with_entrypoint(crate::configure);

tester.tick(); // advance the asset-collection timer

let response = tester.request(/* … */);
```

### `.local_mode()` — local-deployment resilience

Every policy in this repo MUST work in **local mode** (gateway running without a control-plane connection). Without `.local_mode()`, pdk-unit populates the context with fake-but-present Anypoint data (`test_client`, `test_secret`, `anypoint_service_name`, …) — so tests pass even when the policy would panic in a real local deployment.

`.local_mode()` flips `anypoint` to `None`, strips the `ApiContext`, and matches what `HttpPlatformClient::new()` and `StaticPolicyContextCache::read_metadata()` see in disconnected gateways. Default expected behaviour: log a warning, let the request continue (`Flow::Continue`).

Enable the feature in `Cargo.toml`:

```toml
[dev-dependencies]
pdk-unit = { workspace = true, features = ["experimental", "experimental_local_mode"] }
```

Pattern:

```rust
#[test]
fn local_mode_request_passes_through_with_warning() {
    let backend = Rc::new(TraceBackend::new(
        UnitHttpResponse::new(200)
            .with_header("content-type", "application/json")
            .with_body(r#"{"name":"Agent"}"#),
    ));
    let mut tester = UnitTestBuilder::default()
        .local_mode()
        .with_config(r#"{}"#)
        .with_backend(Rc::clone(&backend))
        .with_entrypoint(crate::configure);

    let response = tester.request(
        UnitHttpRequest::get().with_path("/.well-known/agent-card.json"),
    );

    assert!(
        response.status_code() == 200 || response.status_code() == 202,
        "expected pass-through, got {}", response.status_code(),
    );
}
```

If the policy's spec defines a stricter degraded mode (e.g. "structure-only validation when Exchange asset is absent"), test that — but still assert no panic. The authentication object is **not** a local-mode concern; it's set by an upstream auth policy in the filter chain.

## JSON-RPC tests — status code is 200, error lives in the body

Per the JSON-RPC 2.0 contract used across `agent/` and `mcp/`, `response.status_code()` is `200` even when the body carries a JSON-RPC error (`-32700`, `-32600`, `-32602`, …). Always parse the body to assert errors:

```rust
assert_eq!(response.status_code(), 200);
let body: serde_json::Value = serde_json::from_slice(response.body()).unwrap();
assert_eq!(body["error"]["code"], -32602);
```

Exception: when upstream returns a non-2xx with a body, the transcoding policies preserve the upstream status (e.g. 404, 500) while rewriting the body into a JSON-RPC `result` with `isError: true`.

## Upstream verification pattern

When the policy mutates the request before forwarding, assert on the **upstream** request (not on `response.body()` — that holds the upstream's reply):

```rust
let backend = Rc::new(TraceBackend::new(UnitHttpResponse::new(200)));
let mut tester = UnitTestBuilder::default()
    .with_config(config)
    .with_backend(Rc::clone(&backend))
    .with_entrypoint(crate::configure);

tester.request(UnitHttpRequest::post().with_path("/").with_body(body));

let upstream_req = backend.next().unwrap();
assert_eq!(upstream_req.header(":path"), Some("/expected"));
assert_eq!(upstream_req.body(), b"expected body");
```

## UnitTestBuilder Methods

`UnitTestBuilder::default()` is the entry point. All methods return `self` for chaining, except `with_entrypoint` which consumes the builder and returns a `UnitTest`:

| Method | Description |
|--------|-------------|
| `with_config(config)` | Sets policy configuration as a JSON string |
| `with_backend(backend)` | Sets the default HTTP backend. Defaults to a `TraceBackend` returning `200` |
| `with_http_upstream_from_authority(authority, backend)` | Registers an HTTP upstream matched by host authority. Call `metadata()` before this if changing `policy_name` or `policy_namespace` |
| `with_http_upstream(upstream, backend)` | Registers an HTTP upstream by full internal service name |
| `with_grpc_upstream_from_authority(authority, backend)` | Registers a gRPC upstream matched by host authority |
| `with_grpc_upstream(upstream, backend)` | Registers a gRPC upstream by full internal service name |
| `with_identity_management(url, backend)` | Configures identity management service URL and HTTP backend mock for OAuth/identity calls |
| `metadata(f)` | Mutates the `Metadata` injected into policy context. Call before `with_*_upstream_from_authority` methods. Pass `pdk_tests_utils::helpers::configure_asset` for Exchange-fetching policies |
| `local_mode()` | Strips `ApiContext` and sets `anypoint: None` to simulate a control-plane-disconnected deployment. Requires `experimental_local_mode` feature on `pdk-unit` |
| `with_context(f)` | Builds a `UnitTest` from a root-context closure instead of an entrypoint. Use for non-PDK proxy-wasm policies (PDK 1.9+) |
| `with_entrypoint(entrypoint)` | Consumes the builder and creates a `UnitTest`. Pass `crate::configure` as the entrypoint |

## Core API Reference

| Type / Function | Description |
|----------------|-------------|
| `UnitTestBuilder` | Fluent builder. Configure policy, metadata, and backends, then call `with_entrypoint` |
| `UnitTest` | Main orchestrator. Use `request` for synchronous tests or `request_partial` plus `poll` for finer control |
| `UnitHttpRequest` | HTTP request built with method helpers (`get()`, `post()`, etc.) or `custom()`. Use `with_path`, `with_header`, `with_body` |
| `UnitHttpResponse` | HTTP response from `new(status_code)`. Exposes `status_code()` and `UnitHttpMessage` trait accessors |
| `UnitHttpMessage` | Trait for both request and response: `header()`, `headers()`, `body()`, `property()`, `authentication()`, `violation()` |
| `TraceBackend` | Wraps any `Backend`, records each call, assert on upstream requests with `next()` |
| `protobuf_grpc_backend` | Proc-macro that derives `GrpcBackend` from annotated `impl` methods |
| `Backend` | Trait for HTTP upstream mocks. Implemented for closures and structs |
| `GrpcBackend` | Trait for gRPC upstream mocks |
| `UpgradeConnection` | WebSocket handle from `upgrade()` (PDK 1.9+). `response()`, `client()`, `server()` |
| `UnitFrame` | WebSocket frame: `text(payload, fin)`, `binary(payload, fin)`, `ping()`, `pong()`, `close()`; inspect with `frame_type()`, `data()` |
| `UnitFrameType` | Frame-type enum: `Text`, `Binary`, `Ping`, `Pong`, `Close` |

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-unit
- Examples: PDK Custom Policy Examples on GitHub

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-unit.adoc`
- **Snapshot:** 2026-05-14
