---
name: pdk-a2a
description: Use when authoring or reviewing a PDK custom policy that inspects, gates, transforms, validates, or rate-limits A2A (Agent-to-Agent) traffic on Flex Gateway — `message/send`, `message/stream`, `tasks/get`, `tasks/cancel`, `tasks/stream`, `tasks/resubscribe`, `tasks/pushNotificationConfig/*`, `agent/card`, `agent/getAuthenticatedExtendedCard`, plus the v1.0 HTTP+JSON binding (REST-style `/message:send`, `/tasks/{id}:cancel`, etc.) and the v1.0 gRPC binding. Covers protocol-variant detection (Legacy vs V1, JSON-RPC vs HTTP+JSON vs gRPC), the `google.rpc.ErrorInfo` envelope, the A2A method vocabulary and error-code conventions, v1↔v0.3 transcoding, SSE handling for streaming results, and the inspectable-method subset for content-aware policies. Teaches the protocol and the patterns directly so a policy is buildable in any PDK workspace, with or without a shared A2A crate.
---

# PDK A2A

## Overview

A2A (Agent-to-Agent) traffic on Flex Gateway has **three wire bindings**:

1. **JSON-RPC 2.0 over HTTP POST** — Legacy and V1 both use this. The A2A operation is named by the JSON-RPC `method` string; the error lives in-band in the response body.
2. **V1 HTTP+JSON binding** — REST-style URLs (`/message:send`, `/tasks/{id}:cancel`) carrying a bare A2A payload, no JSON-RPC envelope. The operation is named by the (verb, path); errors use native HTTP status + `google.rpc.Status` shape.
3. **V1 gRPC binding** (`application/grpc+proto`, A2A spec §10) — protobuf frames over HTTP/2, status in `grpc-status`/`grpc-message` trailers. A gateway policy can usually only gate it at the transport edge (it can't cheaply parse protobuf), so most content-aware policies pass gRPC through.

A policy that "does something A2A-aware" reduces to: **detect the variant → parse the right envelope (or bare payload) → branch on the operation → optionally inspect typed params → pass through (`Flow::Continue`) or short-circuit with a variant-appropriate error envelope (`Flow::Break`).**

### Portability principle — own the protocol, do not depend on a shared crate

**This skill teaches the A2A protocol contract and the policy patterns directly** — method names, route table, error shapes, the detection algorithm, content locations. With those facts a policy is buildable in **any** PDK workspace using only the `pdk` SDK plus `serde`/`serde_json`. That is the goal: a policy authored from this skill must compile and run in a workspace that has **no** shared A2A library.

Some workspaces happen to ship a shared A2A crate (a workspace library that re-exports method-name constants, a transport classifier, error factories, and SSE helpers). **Reuse such a crate only if it already exists as a dependency in your workspace** — it saves ~150 LOC and tracks the spec. Otherwise inline the small modules in [Pattern B](#pattern-b--self-contained-default). Rules:

- **Never add a workspace shared-crate dependency just to satisfy this skill.** If it isn't already wired in, derive the self-contained modules.
- **The *protocol facts* below are authoritative** (wire strings, codes, routes, content pointers). Any **API/function names** are illustrative — if your workspace's shared crate exposes equivalents, they may be named differently; verify against the crate you actually have.
- Pattern A (reuse a shared crate) and Pattern B (inline ~50 LOC) are **the same protocol** expressed two ways.

## When to Use

- Authoring or editing any policy that recognises A2A JSON-RPC, HTTP+JSON, or gRPC traffic (PII guard / detector, schema validation, prompt decoration, token / call rate limit, agent-card filtering, telemetry, v1↔v0.3 transcoding).
- Reviewing a PR that parses request bodies looking for `"method": "message/send"` (or v1 `"SendMessage"`) or that routes on URL patterns like `/message:send`, `/tasks/{id}:cancel`.
- Adding a new method-specific or operation-specific handler to an existing A2A policy.
- Deciding the JSON-RPC error code / HTTP status / `google.rpc.ErrorInfo` `reason` for an A2A failure.
- Choosing fail-open vs fail-closed posture for non-A2A traffic on the same path.

**Do NOT use** for:

- MCP traffic (`tools/call`, `tools/list`, `resources/*`, `prompts/*`, `initialize`) — see [[pdk-mcp]].
- Generic JSON-RPC outside A2A — use [[pdk-request-headers-bodies]] + [[pdk-stop-execution]] directly.
- Embedding / vector-db work for semantic A2A policies — see [[pdk-embedding-services]] and [[pdk-vector-stores]].

## A2A Method Vocabulary

A2A has **two parallel vocabularies** that name the same operations differently. Pick by variant. Declare the ones you need as `pub const` in your policy (or reuse a shared crate's constants if your workspace has one). Never hard-code the literals at call sites.

### Legacy JSON-RPC (slash-delimited method strings)

These are the authoritative method names for the Legacy JSON-RPC binding.

| Suggested constant | Wire string | Carries body? | Has response? | Inspectable? |
|---|---|---|---|---|
| `MESSAGE_SEND_FUNCTION_NAME` | `message/send` | yes | yes | **yes** |
| `MESSAGE_STREAM_FUNCTION_NAME` | `message/stream` | yes | SSE | **yes** |
| `GET_TASK_FUNCTION_NAME` | `tasks/get` | yes (`id`) | yes | **yes** |
| `CANCEL_TASK_FUNCTION_NAME` | `tasks/cancel` | yes | yes | no |
| `TASKS_STREAM_FUNCTION_NAME` | `tasks/stream` | yes | SSE | **yes** |
| `TASKS_RESUBSCRIBE_FUNCTION_NAME` | `tasks/resubscribe` | yes | SSE | **yes** |
| `SET_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `tasks/pushNotificationConfig/set` | yes | yes | no |
| `GET_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `tasks/pushNotificationConfig/get` | yes | yes | no |
| `LIST_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `tasks/pushNotificationConfig/list` | yes | yes | no |
| `DELETE_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `tasks/pushNotificationConfig/delete` | yes | yes | no |
| `AGENT_CARD_FUNCTION_NAME` | `agent/card` | optional | yes | no |
| `AGENT_CAPABILITIES_FUNCTION_NAME` | `agent/capabilities` | optional | yes | no |
| `AGENT_GET_AUTHENTICATED_EXTENDED_CARD_FUNCTION_NAME` | `agent/getAuthenticatedExtendedCard` | optional | yes | no |

`tasks/list` also exists in the spec but is conventionally **excluded** from the valid-method set on the gateway (no content, no inspection need); add it only if your upstream supports it.

### V1 JSON-RPC (CamelCase / proto-style method strings)

V1 reuses JSON-RPC 2.0 but renames methods to a CamelCase vocabulary aligned with `a2a.proto v1.0.0`:

| Suggested constant | Wire string |
|---|---|
| `MESSAGE_SEND_FUNCTION_NAME` | `SendMessage` |
| `SEND_STREAMING_MESSAGE_FUNCTION_NAME` | `SendStreamingMessage` |
| `GET_TASK_FUNCTION_NAME` | `GetTask` |
| `LIST_TASKS_FUNCTION_NAME` | `ListTasks` |
| `CANCEL_TASK_FUNCTION_NAME` | `CancelTask` |
| `SUBSCRIBE_TASK_FUNCTION_NAME` | `SubscribeToTask` |
| `CREATE_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `CreateTaskPushNotificationConfig` |
| `GET_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `GetTaskPushNotificationConfig` |
| `LIST_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `ListTaskPushNotificationConfigs` |
| `DELETE_PUSH_NOTIFICATION_TASK_FUNCTION_NAME` | `DeleteTaskPushNotificationConfig` |
| `AGENT_GET_EXTENDED_CARD_FUNCTION_NAME` | `GetExtendedAgentCard` |

### V1 HTTP+JSON binding (REST routes, no JSON-RPC envelope)

| Verb | Bare path | Operation | Body? |
|---|---|---|---|
| POST | `/message:send` | `SendMessage` | yes |
| POST | `/message:stream` | `SendStreamingMessage` | yes (SSE response) |
| GET | `/tasks` | `ListTasks` | no |
| GET | `/tasks/{id}` | `GetTask` | no |
| POST | `/tasks/{id}:cancel` | `CancelTask` | yes |
| POST | `/tasks/{id}:subscribe` | `SubscribeToTask` | yes (SSE response) |
| POST | `/tasks/{taskId}/pushNotificationConfigs` | `CreateTaskPushNotificationConfig` | yes |
| GET | `/tasks/{taskId}/pushNotificationConfigs` | `ListTaskPushNotificationConfigs` | no |
| GET | `/tasks/{taskId}/pushNotificationConfigs/{id}` | `GetTaskPushNotificationConfig` | no |
| DELETE | `/tasks/{taskId}/pushNotificationConfigs/{id}` | `DeleteTaskPushNotificationConfig` | no |
| GET | `/extendedAgentCard` | `GetExtendedAgentCard` | no |

Tenant-prefixed alternates (`/{tenant}/<bare>`) classify to the same operation; the tenant value is not extracted. The defined routes use `POST`/`GET`/`DELETE`; the binding reserves `PUT` as a REST verb too (no current route uses it). There is no `PATCH`.

### Inspectable methods (content-bearing subset)

Content-aware policies (PII detection, prompt decoration, token rate limit) only inspect methods that carry **user-authored content**:

```
message/send, message/stream, tasks/get, tasks/stream, tasks/resubscribe
```

(V1 equivalents: `SendMessage`, `SendStreamingMessage`, `GetTask`, `SubscribeToTask`.) Other methods (`agent/card`, `tasks/cancel`, push-config ops, `ListTasks`) carry only IDs/config and **always pass through**.

## Protocol-Variant Detection

The single biggest pitfall in A2A policies is **picking the wrong variant** and serialising the wrong error shape. Detection is two steps.

### Step 1 — header/path-only (provisional, before reading body)

V1 is signalled by **either**:
- `A2A-Version: 1.0` request header (match the header name case-insensitively), **or**
- `?A2A-Version=1.0` query parameter (case-insensitive key).

```rust
// Self-contained detection — no shared crate required.
fn is_a2a_v1(path: &str, get_header: impl Fn(&str) -> Option<String>) -> bool {
    if get_header("A2A-Version").map(|v| v.trim() == "1.0").unwrap_or(false) {
        return true;
    }
    // query param, case-insensitive key
    path.split_once('?')
        .map(|(_, q)| q.split('&').any(|kv| {
            let (k, v) = kv.split_once('=').unwrap_or((kv, ""));
            k.eq_ignore_ascii_case("a2a-version") && v == "1.0"
        }))
        .unwrap_or(false)
}
```

Header/path detection alone yields a **provisional** variant of `Legacy` or `V1(JsonRpc)`; HTTP+JSON can only be confirmed after seeing the body (or the verb).

### Step 2 — body-aware (definitive)

Classify the transport from `(method, is_v1, body)`:

- **POST + body containing a `"jsonrpc":"2.0"` envelope ⇒ JSON-RPC** (Legacy or V1 by `is_v1`).
- **POST without a JSON-RPC envelope, `is_v1` true ⇒ HTTP+JSON.**
- **GET / PUT / DELETE, `is_v1` true ⇒ HTTP+JSON.**
- **`Content-Type: application/grpc*` ⇒ gRPC** (transport-edge only; usually pass through).
- **Anything else ⇒ not A2A traffic — pass through (`Flow::Continue`).**

```rust
enum Transport { JsonRpc, HttpJson, Grpc }
enum Variant { Legacy, V1(Transport) }

fn classify(method: &str, is_v1: bool, content_type: Option<&str>, body: &[u8]) -> Option<Variant> {
    if content_type.map(|c| c.starts_with("application/grpc")).unwrap_or(false) {
        return Some(Variant::V1(Transport::Grpc)); // only meaningful in V1
    }
    let looks_jsonrpc = body.windows(9).any(|w| w == b"\"jsonrpc\"");
    match (method, looks_jsonrpc, is_v1) {
        ("POST", true,  true)  => Some(Variant::V1(Transport::JsonRpc)),
        ("POST", true,  false) => Some(Variant::Legacy),
        ("POST", false, true)  => Some(Variant::V1(Transport::HttpJson)),
        ("GET" | "PUT" | "DELETE", _, true) => Some(Variant::V1(Transport::HttpJson)),
        _ => None,
    }
}
```

(If your workspace has a shared classifier, use it instead — but the rules above are what it must implement.)

### Threading the variant to the response side

Stash the detected variant in a stream property so the response filter knows which error shape to use when rewriting upstream errors. See [[pdk-share-data-request-response]].

```rust
stream_properties.set_property(&["a2a", "variant"], Some(match variant {
    Variant::V1(Transport::HttpJson) => b"http+json" as &[u8],
    Variant::V1(Transport::JsonRpc)  => b"jsonrpc"   as &[u8],
    Variant::V1(Transport::Grpc)     => b"grpc"      as &[u8],
    Variant::Legacy                  => b"legacy"    as &[u8],
}));
```

## Error Envelopes — Three Shapes, Same Information

A2A has **three** wire shapes for errors. Pick by variant; **never mix**.

### Shape 1 — Legacy JSON-RPC

`data` is a plain string. HTTP status is **200** (JSON-RPC convention: the transport succeeded, the failure is in-band).

```json
{
  "jsonrpc": "2.0",
  "id": <original-id, or null>,
  "error": { "code": -32602, "message": "Invalid parameters", "data": "Missing `tasks/get` params" }
}
```

### Shape 2 — V1 JSON-RPC (`google.rpc.ErrorInfo` in `data`)

Same JSON-RPC envelope; `data` is a **single-element array** carrying a typed `ErrorInfo`. HTTP status **200**.

```json
{
  "jsonrpc": "2.0",
  "id": <original-id, or null>,
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": [{
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "METHOD_NOT_FOUND",
      "domain": "a2a-protocol.org",
      "metadata": { "detail": "Invalid methods: `Foo`. Valid Methods: ..." }
    }]
  }
}
```

### Shape 3 — V1 HTTP+JSON (`google.rpc.Status` shape, native HTTP status)

No JSON-RPC envelope. HTTP status matches `error.code`. `details` carries the typed `ErrorInfo`.

```json
{
  "error": {
    "code": 404,
    "message": "Method not found",
    "details": [{
      "@type": "type.googleapis.com/google.rpc.ErrorInfo",
      "reason": "METHOD_NOT_FOUND",
      "domain": "a2a-protocol.org",
      "metadata": { "detail": "Invalid methods: `POST /unknown`. Valid Methods: ..." }
    }]
  }
}
```

> **HTTP status caveat.** The protocol convention for both JSON-RPC shapes is **HTTP 200** with the error in-band. Some shared-crate helpers return **HTTP 400** for envelope-level parse / invalid-request failures (the parse failed before a valid `id` existed). Both are seen in the wild; **200 is the spec-faithful default**. If you reuse a shared crate's error-response builder, verify which status it emits and stay consistent within your policy.

### Conventional codes / reasons

The numeric range `-32001..-32009` is **A2A-specific and means different things in Legacy vs V1** — this is the most common error-table mistake.

| Condition | JSON-RPC `code` | HTTP status (HTTP+JSON) | `ErrorInfo.reason` |
|---|---|---|---|
| Parse error (body not JSON) | `-32700` | 400 | `PARSE_ERROR` |
| Invalid request (bad envelope) | `-32600` | 400 | `INVALID_REQUEST` |
| Method not found | `-32601` | 404 | `METHOD_NOT_FOUND` |
| Invalid params | `-32602` | 400 | `INVALID_PARAMS` |
| Internal error | `-32603` | 500 | `INTERNAL` |
| Task not found | `-32001` | — | `TASK_NOT_FOUND` |
| Task not cancellable | `-32002` | — | `TASK_NOT_CANCELLABLE` |
| **V1:** push notification not supported | `-32003` | — | `PUSH_NOTIFICATION_NOT_SUPPORTED` |
| **V1:** unsupported operation | `-32004` | — | `UNSUPPORTED_OPERATION` |
| **V1:** content type not supported | `-32005` | 400 | `CONTENT_TYPE_NOT_SUPPORTED` |
| **V1:** invalid agent response | `-32006` | — | `INVALID_AGENT_RESPONSE` |
| **V1:** extended card not configured | `-32007` | — | `AUTHENTICATED_EXTENDED_CARD_NOT_CONFIGURED` |
| **V1:** extension support required | `-32008` | — | `EXTENSION_SUPPORT_REQUIRED` |
| **V1:** version not supported | `-32009` | — | `VERSION_NOT_SUPPORTED` |
| Rate limit exceeded | (custom) | 429 | `RESOURCE_EXHAUSTED` |

**Legacy auth-error caveat.** In Legacy, the same `-32003`/`-32004` numbers were historically used for "authentication failed" / "unauthorized access" — but real-world error factories often map authentication → **`-32007`** and forbidden → **`-32008`** instead, and tests that exercise `unauthorized()`/`forbidden()` constructors confirm those numbers on the wire. The numbers are **not stable across A2A versions and implementations** — for auth failures prefer the HTTP+JSON native statuses (401/403) where you control the binding, and when you must emit a JSON-RPC code, pick the value your downstream clients actually branch on and document it. Do not assume `-32003`=auth in V1; there it means `PUSH_NOTIFICATION_NOT_SUPPORTED`.

Always preserve the original request `id` (use `null` if it can't be recovered) and `jsonrpc: "2.0"` for the envelope shapes.

## Pattern A — Reuse a shared crate (only if your workspace already has one)

If — and only if — your workspace **already** depends on a shared A2A library (method-name constants, a transport classifier, error factories, SSE helpers), reuse it instead of re-deriving the tables above. Typical shape (names will vary — verify against your crate):

```toml
[dependencies]
pdk        = { workspace = true, features = ["experimental"] }
serde      = { workspace = true }
serde_json = { workspace = true }
# plus your workspace's shared A2A crate, IF it is already a dependency
```

A shared crate usually gives you: method-name constants for both vocabularies; `is_a2a_v1` / provisional-variant / body-aware classifiers; an HTTP+JSON URL→operation router with a `valid_routes()` list and per-operation body validators; three error factories (Legacy string-`data`, V1 `ErrorInfo`-`data`, HTTP+JSON `details`-array) plus a variant-aware factory that takes `(variant, code, reason, detail)` and picks the shape; a `create_*_error_response(...)` that wraps an error into a `Response` and (optionally) fires a policy violation; and SSE decode helpers. **Do not add such a crate as a new dependency just to get these** — derive them inline (Pattern B). When a shared crate is wrapped behind a facade module (e.g. one crate re-exporting another's `a2a` / `a2a_v1` / `json_rpc` submodules), confirm the real import path before relying on it.

The request-filter control flow is identical to Pattern B; only the helper calls differ.

## Pattern B — Self-contained (default)

The portable default: ship a small per-crate `a2a.rs` (method names + inspectable subset) and `jsonrpc.rs` (borrowed parser + envelope). ~50 LOC, no external A2A dependency. Extract these into a shared crate only once a **third** policy in the same workspace needs them (YAGNI) — and even then, only with the team's agreement.

### `a2a.rs` — method names + inspectable subset

```rust
//! A2A JSON-RPC method names. List the methods this policy acts on; others pass through.

pub const GET_TASK_FUNCTION_NAME: &str       = "tasks/get";
pub const TASKS_STREAM_FUNCTION_NAME: &str   = "tasks/stream";
pub const MESSAGE_SEND_FUNCTION_NAME: &str   = "message/send";
pub const MESSAGE_STREAM_FUNCTION_NAME: &str = "message/stream";
pub const TASKS_RESUBSCRIBE_FUNCTION_NAME: &str = "tasks/resubscribe";

pub fn is_inspectable_method(method: &str) -> bool {
    matches!(
        method,
        MESSAGE_SEND_FUNCTION_NAME
            | MESSAGE_STREAM_FUNCTION_NAME
            | GET_TASK_FUNCTION_NAME
            | TASKS_STREAM_FUNCTION_NAME
            | TASKS_RESUBSCRIBE_FUNCTION_NAME
    )
}
```

To support both Legacy and V1 wire names, declare the V1 set too (`SendMessage`, `SendStreamingMessage`, `GetTask`, `SubscribeToTask`) and union them in `is_inspectable_method`.

### `jsonrpc.rs` — borrowed parser + envelope

```rust
use serde::{Deserialize, Serialize};
use serde_json::value::RawValue;
use serde_json::Value;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum JsonRpcId { String(String), Int(i64), Uint(u64) }

pub fn parse_jsonrpc_id(raw: &RawValue) -> Option<JsonRpcId> {
    let v: Value = serde_json::from_str(raw.get()).ok()?;
    match v {
        Value::String(s) => Some(JsonRpcId::String(s)),
        Value::Number(n) => n.as_i64().map(JsonRpcId::Int)
            .or_else(|| n.as_u64().map(JsonRpcId::Uint)),
        _ => None,
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JsonRpcRequest<'a> {
    pub method: &'a str,
    pub params: Option<&'a RawValue>,
    pub id:     Option<&'a RawValue>,
    pub jsonrpc: Option<&'a str>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JsonRpcResponse {
    pub jsonrpc: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")] pub id: Option<JsonRpcId>,
    #[serde(skip_serializing_if = "Option::is_none")] pub result: Option<Value>,
    #[serde(skip_serializing_if = "Option::is_none")] pub error: Option<RpcError>,
}

#[derive(Clone, Debug, Deserialize, Serialize)]
pub struct RpcError {
    pub code: i32,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")] pub data: Option<Value>,
}

impl JsonRpcResponse {
    pub fn error(id: Option<JsonRpcId>, error: RpcError) -> Self {
        Self { result: None, error: Some(error), id, jsonrpc: Some("2.0".to_string()) }
    }
}
```

Use **borrowed** `&str` / `&RawValue` to avoid copying the body.

### Request filter (Legacy JSON-RPC, content inspection)

```rust
use pdk::hl::*;

async fn request_filter(rs: RequestState, /* pv, sp, ... */) -> Flow<()> {
    let header_state = rs.into_headers_state().await;
    let h = header_state.handler();
    if header_state.method().as_str() != "POST" { return Flow::Continue(()); }

    let ct = match h.header("content-type") { Some(c) => c, None => return Flow::Continue(()) };
    if !ct.starts_with("application/json") { return Flow::Continue(()); }

    let body_state = header_state.into_body_state().await;
    let body = body_state.as_bytes();
    let body = body.as_slice();

    let req: JsonRpcRequest<'_> = match serde_json::from_slice(body) {
        Ok(r) => r,
        Err(_) => return Flow::Continue(()),       // not A2A JSON-RPC → fail-open
    };
    if !is_inspectable_method(req.method) { return Flow::Continue(()); }

    let request_id = req.id.and_then(parse_jsonrpc_id);
    let params: Value = match req.params.and_then(|p| serde_json::from_str(p.get()).ok()) {
        Some(v) => v,
        None    => return Flow::Continue(()),
    };

    inspect_payload(&params, request_id /* ... */)
}
```

For a variant-aware filter that also handles HTTP+JSON, branch on `classify(...)` (above) and route HTTP+JSON requests on `(verb, path)` against the route table, validating the body per operation; emit Shape 3 errors for HTTP+JSON and Shape 1/2 for JSON-RPC.

### Walking the payload — canonical content locations

A2A content lives at well-known JSON pointers. Inspect the union; non-existent ones short-circuit cheaply.

| Pointer | Method that produces it |
|---|---|
| `/message/parts` | `message/send`, `message/stream`, `tasks/resubscribe` |
| `/task/parts`    | `tasks/stream` |
| `/task/description` | `tasks/get`, some `tasks/*` ops |

Each `parts[i]` is `{ "kind"|"type": "text", "text": "<user content>" }`. **Accept either discriminator spelling** — legacy parts use `type`, current parts use `kind`.

```rust
if let Some(Value::Array(parts)) = json.pointer("/message/parts") { scan_parts(parts, /* ... */); }
if let Some(Value::Array(parts)) = json.pointer("/task/parts")    { scan_parts(parts, /* ... */); }
if let Some(Value::String(d))    = json.pointer("/task/description") { scan_text(d, /* ... */); }
```

### Error response — JSON-RPC envelope at HTTP 200

```rust
pub fn create_a2a_error_response(request_id: Option<JsonRpcId>, err: RpcError) -> Response {
    let resp = JsonRpcResponse::error(request_id, err);
    let body = serde_json::to_string(&resp).unwrap_or_default();
    Response::new(200)                                              // ← JSON-RPC convention
        .with_headers(vec![("content-type".to_string(), "application/json".to_string())])
        .with_body(body.into_bytes())
}
```

Self-contained policies usually target only the JSON-RPC binding (Legacy or V1). If you also need HTTP+JSON, mirror Shape 3 exactly (native HTTP status, `details` array) so behaviour stays consistent.

## v1 ↔ v0.3 Transcoding

A2A v1 and the older v0.3 are **the same JSON-RPC envelope with different method names, params, enums, results, and error reasons**. A transcoding policy sits between a v1 client and a v0.3 upstream (or vice-versa) and rewrites on both sides:

- **Method name**: `SendMessage` ⇄ `message/send`, `GetTask` ⇄ `tasks/get`, `SubscribeToTask` → `tasks/resubscribe`, `GetExtendedAgentCard` → `agent/getAuthenticatedExtendedCard`, etc.
- **Gate v1-only methods that v0.3 can't serve**: `ListTasks` → reject `-32601` (method unsupported for v0.3); the four push-config methods → reject `-32003 PUSH_NOTIFICATION_NOT_SUPPORTED` before forwarding.
- **Params / results / enums**: field renames and shape changes per the v1↔v0.3 mapping.
- **SSE events**: transcode each streamed chunk, not just the unary response.
- **Agent card**: transcode the card document shape.
- **Errors**: map v0.3 error codes back to v1 `ErrorInfo` `reason` tags.

Transcoding is a pure-rewrite policy — it never inspects user content, so it touches **every** method, not just the inspectable subset. Keep the full field map in one place (a `spec.md` alongside the implementation works well).

## SSE Responses (`text/event-stream`)

Streaming methods — `message/stream`, `tasks/stream`, `tasks/resubscribe` (Legacy) and `SendStreamingMessage`, `SubscribeToTask` (V1) — return SSE, one `message` event per JSON-RPC chunk. **Buffering breaks the streaming contract**; stream-rewrite or skip.

Decode SSE with a small adapter (or your workspace's SSE helper if it has one):

```rust
use futures::StreamExt;
use pdk::hl::BodyStream;

pub fn into_sse(stream: BodyStream<'_>)
    -> impl Stream<Item = Result<async_sse::Event, SseError>> + use<'_>
{
    use futures::TryStreamExt;
    let read = stream.map(|chunk| Ok(chunk.into_bytes())).into_async_read();
    async_sse::decode(read).map(|i| i.map_err(|_| SseError::Decode))
}
```

Branch on the response content-type:

```rust
let resp_ct = response_headers.handler().header("content-type").unwrap_or_default();
if resp_ct.starts_with("application/json") {
    // buffer + parse one JsonRpcResponse, inspect result
} else if resp_ct.starts_with("text/event-stream") {
    // stream via into_sse; each `message` event is one JsonRpcResponse JSON
} else {
    // not an A2A response we know how to read — pass through
}
```

For streaming where bytes have already left the gateway, **debit-after-the-fact**: the current response passes through; if a limit is exceeded *during* response accounting, drain the bucket so the **next** request is blocked. **Disable the upstream timeout** before entering body state when you expect SSE, or long-running tasks get cut off — use the PDK request handler's no-timeout control (see [[pdk-request-headers-bodies]] / [[pdk-timer]]).

## Traffic Recognition

A request is "A2A traffic" iff one of these holds:

1. **Legacy / V1 JSON-RPC**: `POST` + `Content-Type: application/json` + body is a JSON-RPC 2.0 envelope (contains `"jsonrpc":"2.0"`) + method matches the A2A vocabulary.
2. **V1 HTTP+JSON**: `is_v1` (header or query) **and** `(POST + non-JSON-RPC body)` or `(GET | PUT | DELETE)` **and** URL matches an HTTP+JSON route.
3. **V1 gRPC**: `is_v1` and `Content-Type: application/grpc*` (transport-edge handling only).

**Everything else passes through (fail-open):** non-POST without a v1 tag, non-JSON content type, malformed JSON, JSON-RPC with a non-A2A method, HTTP+JSON URL not in the route table. Log at `debug` on parse failure. **Never** return an A2A error envelope for traffic that isn't A2A — the policy may be one of several on the chain and the request might not be intended for an A2A server.

Exception: if the policy's own configuration (e.g. PII detector) rejects a **recognised** A2A request, that is a **fail-closed** decision — see below.

## Local-Mode and Failure-Mode Contract

- The policy MUST load and serve from t=0 even when the control plane is unreachable.
- No `unwrap()` / `panic!` on context-derived `Option`/`Result`. Use `if let`, `match`, or `unwrap_or` with a logged warning.
- Test the no-control-plane path with the PDK unit harness's local-mode builder (see [[pdk-unit-tests]]).

| Condition | Posture | Why |
|---|---|---|
| Body parse fails / non-A2A traffic | **fail-open** (`Flow::Continue`), debug log | request might not be intended for an A2A server |
| Unknown JSON-RPC method (in A2A envelope) | reject with `-32601 / METHOD_NOT_FOUND` (V1 / Legacy) or HTTP 404 (HTTP+JSON) | the envelope says A2A, but the method isn't A2A |
| Required policy config evaluation fails | **fail-closed**, `-32600 / INVALID_REQUEST` (or HTTP 400 for HTTP+JSON) | a misconfigured policy that allows traffic is the wrong default |
| Detector / external dependency failure (PII engine, IdP, contract service) | **spec-decides** | PII detection failing-open is almost certainly wrong; telemetry failing-closed is almost certainly wrong. The spec must name the failure mode and defend it. |
| Limit exceeded | **fail-closed**, `429 / RESOURCE_EXHAUSTED` | by definition |
| Streaming response bytes already on the wire | **debit-after-the-fact**, block next request | can't recall bytes already streamed |
| Schema drift on response (typed `R` doesn't deserialise) | **pass through**, no handler invoked | upstream schema drift shouldn't crash the filter |

## Capability Checklist

Whether you derive these (Pattern B) or get them from a shared crate (Pattern A), an A2A-aware policy needs:

| Capability | What it does |
|---|---|
| V1 detection from header / query | `A2A-Version: 1.0` header or `?A2A-Version=1.0` query |
| provisional variant (pre-body) | `Legacy` vs `V1(JsonRpc)` from headers/path |
| body-aware transport classification | distinguish JSON-RPC / HTTP+JSON / gRPC / not-A2A |
| HTTP+JSON URL → operation router | `(verb, path)` → operation; plus a `valid_routes()` list for error detail |
| per-operation body validation | typed param validation for the matched operation |
| method validation | reject unknown method names early; expose the valid-method list for error detail |
| three error factories | Legacy (string `data`), V1 JSON-RPC (`ErrorInfo` `data`), HTTP+JSON (`details` array, native status) |
| variant-aware error factory | `(variant, code, reason, detail)` → correct shape |
| error→`Response` wrapper | build the `Response` and fire a policy violation on deny ([[pdk-policy-violations]]) |
| SSE decode | adapter over `BodyStream` for streaming responses |

## Examples

### Inspectable-method content scan (self-contained, Legacy)

A self-contained request filter (no shared crate) decodes the A2A message, scans the inspectable-method content, and blocks or forwards — all in one `request.rs`.

### Variant-aware schema validation

A request filter that handles all three variants branches on the detected variant up front, then validates the request body against the matching JSON schema before forwarding. Keep the variant detection and schema-selection logic in one place so the three paths stay in sync.

### Per-(caller, method) rate-limit `keySelector` (operator config)

```yaml
maximumRequests: 60
timePeriodInMilliseconds: 60000
keySelector: "#[attributes.headers['client_id'] ++ '|' ++ vars.a2a.method]"
```

Set `vars.a2a.method` from the parsed JSON-RPC envelope on the request side **before** evaluating the `keySelector`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Returning a Legacy `data: "<string>"` error for a V1 request | V1 `data` must be a `[ErrorInfo]` array |
| Returning a JSON-RPC envelope for an HTTP+JSON request | use Shape 3 — native HTTP status, `details` array, no JSON-RPC envelope |
| Returning HTTP 4xx/5xx for a JSON-RPC request | JSON-RPC convention is HTTP 200 with the error in the body (some helpers use 400 for envelope parse failures — pick one and be consistent) |
| Assuming `-32003` means "auth failed" everywhere | in V1 it means `PUSH_NOTIFICATION_NOT_SUPPORTED`; the auth codes are not stable — prefer native 401/403 in HTTP+JSON |
| Hard-coding `"message/send"` / `"SendMessage"` literals at call sites | declare `pub const` once per crate (or reuse a shared crate's constants); literals drift between Legacy and V1 |
| Detecting variant from path/headers only | header-only is **provisional**; refine with body-aware classification |
| Treating non-POST as non-A2A | V1 HTTP+JSON uses GET / PUT / DELETE — gate on `is_v1` instead |
| Returning an A2A error for non-A2A traffic | fail-open — `Flow::Continue` with a debug log |
| Buffering an SSE response | stream-rewrite or skip |
| Trying to recall bytes already streamed | not possible — debit-after-the-fact, block the *next* request |
| Inspecting `tasks/cancel` or push-config methods for content | only inspect `message/*` + `tasks/get` + `tasks/stream` + `tasks/resubscribe` (Legacy) / `Send*Message`, `GetTask`, `SubscribeToTask` (V1) |
| Calling `unwrap()` on a context-derived `Option` | local-mode must not panic; use `if let` / `match` / `unwrap_or` and log |
| Forgetting to disable the upstream timeout for SSE methods | disable it before entering body state |
| Mixing the `kind`/`type` part discriminator | accept both — legacy parts use `type`, current parts use `kind` |
| Not threading the variant from request to response | stash `vars.a2a.variant` so the response filter picks the right error shape |
| Adding a shared A2A crate just to get the constants | derive Pattern B inline; only adopt a shared crate that already exists in the workspace |

## Red Flags

- "I'll just buffer the SSE response so I can rewrite it" — no; stream-rewrite or skip.
- "I'll panic if the body isn't valid JSON" — no; debug log and `Flow::Continue` (fail-open) for non-A2A traffic.
- "I'll match `request.method` against literal strings" — declare `pub const` once (or reuse existing constants); literals drift between Legacy and V1.
- "I'll return HTTP 400 for every JSON-RPC failure" — JSON-RPC errors are in-band at HTTP 200 by spec; only HTTP+JSON uses native statuses.
- "I'll add a shared A2A crate dependency to get these helpers" — derive them inline unless the crate is **already** a workspace dependency.
- "I'll detect V1 from the path only" — path-only is provisional; an HTTP+JSON request may need body inspection.
- "I'll inspect `tasks/cancel` payloads for PII" — push-config and cancel methods don't carry user content; pass through.
- "`-32003` is the auth-failed code" — only in Legacy historically, and even there it's often `-32007`; in V1 it's `PUSH_NOTIFICATION_NOT_SUPPORTED`.

## Cross-References

- [[pdk-create-policy]] — split-model crate layout for new A2A policies.
- [[pdk-mcp]] — companion skill for MCP traffic; envelope-shape and SSE patterns are parallel.
- [[pdk-rate-limiting]] — rate-limit instance, `is_allowed`, clustered storage (used by token-rate-limit-style A2A policies).
- [[pdk-request-headers-bodies]] — reading request/response bodies and controlling the upstream timeout.
- [[pdk-stop-execution]] — short-circuiting with `Flow::Break(response)`.
- [[pdk-policy-violations]] — required when blocking on a deny decision (rate exceeded, schema invalid, PII detected).
- [[pdk-share-data-request-response]] — threading the A2A variant (and method) from request to response filter.
- [[pdk-dataweave]] — evaluating operator-supplied `keySelector` expressions referencing `vars.a2a.*`.
- [[pdk-schema-definition]] — defining `keySelector`, `maximumRequests`, etc. in `gcl.yaml`.
- [[pdk-unit-tests]] — unit harness + local-mode for the no-control-plane path.
- [[pdk-integration-tests]] — integration recipes for SSE responses, JSON-RPC envelope assertions, HTTP+JSON URL routing.
