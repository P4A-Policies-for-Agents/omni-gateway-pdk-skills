---
name: pdk-mcp
description: Use when authoring or reviewing a PDK custom policy that inspects, gates, transforms, validates, routes, or optimizes MCP (Model Context Protocol) traffic on Flex Gateway — `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`, `resources/templates/list`, `prompts/list`, `prompts/get`, `completion/complete`, `logging/setLevel`, `ping`, and `notifications/*`. Covers the single JSON-RPC 2.0 vocabulary, Streamable-HTTP transport with SSE responses, the JSON-RPC error envelope and code conventions, the request→response method-threading problem, tool/resource/prompt content locations, access control, tool-name mapping, payload optimization, and MCP↔HTTP transcoding. Teaches the protocol and the patterns directly so a policy is buildable in any PDK workspace, with or without a shared MCP crate.
---

# PDK MCP

## Overview

MCP (Model Context Protocol) traffic on Flex Gateway is **JSON-RPC 2.0 over HTTP POST** (`application/json`). Responses arrive as **either** a single JSON body **or** an SSE stream (`text/event-stream`) where each `data:` frame is one JSON-RPC response. Unlike A2A, MCP has **one method vocabulary and one transport family** — there is no Legacy/V1 split and no separate REST binding — so variant detection is trivial and the work is mostly: **parse the JSON-RPC envelope → branch on `method` → optionally inspect typed `params` → pass through (`Flow::Continue`) or short-circuit with a JSON-RPC error envelope (`Flow::Break`).**

What makes MCP *harder* than A2A is the **request→response asymmetry**: the response carries no method name, so a response-side policy must either thread the method from the request side or infer it from the response shape; and the content a policy cares about (tool lists, resource lists, prompt lists) often lives on the **response** side, in SSE streams.

### Portability principle — own the protocol, do not depend on a shared crate

**This skill teaches the MCP protocol contract and the policy patterns directly** — the method vocabulary, content locations, error shape, transport handling. With those facts a policy is buildable in **any** PDK workspace using only the `pdk` SDK plus `serde`/`serde_json`. A policy authored from this skill must compile and run in a workspace that has **no** shared MCP library.

Some workspaces happen to ship a shared MCP crate (method-name constants, a typed dispatcher framework, error factories, SSE handling, token-metric helpers) — often layered on a generic JSON-RPC crate and an off-the-shelf MCP schema crate. **Reuse such a crate only if it already exists as a dependency in your workspace.** Otherwise inline the small modules in [Pattern B](#pattern-b--self-contained-default). Rules:

- **Never add a workspace shared-crate dependency just to satisfy this skill.** Derive the self-contained modules instead.
- **The *protocol facts* below are authoritative** (method strings, content pointers, error codes, transport behaviour). Any **API/function/type names** are illustrative — verify against the crate you actually have.
- Pattern A (reuse a shared crate / dispatcher) and Pattern B (inline parse + branch) are **the same protocol** expressed two ways.

A minimal "smallest MCP policy" (just enables SSE) lives in the public examples repo at `pdk-custom-policy-examples/agent-policies/mcp-support-policy/` — start there for the SSE-enable pattern.

## When to Use

- Authoring or editing any policy that recognises MCP traffic: schema validation, access control / authz (identity-aware or not), tool-name mapping, payload / token optimization, PII detection, MCP↔HTTP transcoding, upstream routing, telemetry.
- Reviewing a PR that parses request bodies looking for `"method":"tools/call"` (or `tools/list`, `resources/read`, `prompts/get`, `initialize`) or that filters `*/list` response arrays.
- Deciding the JSON-RPC error code and HTTP status for an MCP failure.
- Handling the request→response method-threading or SSE-streaming problem for MCP.

**Do NOT use** for:

- A2A traffic (`message/send`, `tasks/*`, `agent/card`) — see [[pdk-a2a]].
- Generic JSON-RPC outside MCP — use [[pdk-request-headers-bodies]] + [[pdk-stop-execution]] directly.
- Embedding / vector-db work for semantic MCP policies — see [[pdk-embedding-services]] and [[pdk-vector-stores]].

## MCP Method Vocabulary

One JSON-RPC vocabulary. Declare the methods you act on as `pub const` (or reuse a shared crate's constants). Never hard-code literals at call sites.

### Client requests (have a response)

| Suggested constant | Wire string | Typed params | Inspectable side |
|---|---|---|---|
| `INITIALIZE_METHOD_NAME` | `initialize` | handshake (`protocolVersion`, `capabilities`, `clientInfo`) | — (handshake) |
| `PING_METHOD_NAME` | `ping` | none | — |
| `TOOLS_LIST_METHOD_NAME` | `tools/list` | pagination cursor | **response** (`result.tools[]`) |
| `TOOLS_CALL_METHOD_NAME` | `tools/call` | `{ name, arguments }` | **request** (`params.name`, `params.arguments`) + response |
| `RESOURCES_LIST_METHOD_NAME` | `resources/list` | pagination cursor | **response** (`result.resources[]`) |
| `RESOURCES_TEMPLATES_LIST_METHOD_NAME` | `resources/templates/list` | pagination cursor | **response** (`result.resourceTemplates[]`) |
| `RESOURCES_READ_METHOD_NAME` | `resources/read` | `{ uri }` | **request** (`params.uri`) + response |
| `RESOURCES_SUBSCRIBE_METHOD_NAME` | `resources/subscribe` | `{ uri }` | — |
| `RESOURCES_UNSUBSCRIBE_METHOD_NAME` | `resources/unsubscribe` | `{ uri }` | — |
| `PROMPTS_LIST_METHOD_NAME` | `prompts/list` | pagination cursor | **response** (`result.prompts[]`) |
| `PROMPTS_GET_METHOD_NAME` | `prompts/get` | `{ name, arguments }` | **request** (`params.name`) + response |
| `LOGGING_SET_LEVEL_METHOD_NAME` | `logging/setLevel` | `{ level }` | — |
| `COMPLETION_COMPLETE_METHOD_NAME` | `completion/complete` | `{ ref, argument }` | maybe |

> Naming caveat: a shared crate may name the `prompts/get` constant `PROMPTS_GET_*` in one workspace and `PROMPTS_READ_*` in another. Match the **wire string** `prompts/get`, not the constant identifier.

Task-style methods (`tasks/get`, `tasks/cancel`, `tasks/list`, `tasks/result`) appear in some MCP server profiles; they carry only IDs/config and pass through for content-aware policies.

### Client notifications (NO response, NO `id`)

Notifications are JSON-RPC messages **without an `id`**. They are fire-and-forget — **never emit an error response to a notification.** Detect them by the `notifications/` method prefix.

| Suggested constant | Wire string |
|---|---|
| `NOTIFICATION_INITIALIZED_METHOD_NAME` | `notifications/initialized` |
| `NOTIFICATION_CANCELLED_METHOD_NAME` | `notifications/cancelled` |
| `NOTIFICATION_PROGRESS_METHOD_NAME` | `notifications/progress` |
| `NOTIFICATION_ROOTS_LIST_CHANGED_METHOD_NAME` | `notifications/roots/list_changed` |

### Inspectable / content-bearing subset

- **Request side**: `tools/call` (`params.name`, `params.arguments`), `resources/read` (`params.uri`), `prompts/get` (`params.name`).
- **Response side**: the `*/list` methods carry content in their results — `result.tools[]`, `result.resources[]`, `result.resourceTemplates[]`, `result.prompts[]` — and `tools/call` returns `result.content[]` (each `{ type, text }`).

Access-control, mapping, optimization, and PII policies therefore almost always need **both** a request filter and a response filter.

## Transport / Recognition

There is no multi-variant classifier like A2A. MCP recognition is:

1. **Request side** — a request is MCP iff: `POST` + JSON content-type + body parses as a `jsonrpc:"2.0"` envelope + method is in the MCP vocabulary. Malformed JSON → JSON-RPC `-32700`; missing/invalid `jsonrpc` field → `-32600`.
2. **Response side** — dispatch by **Content-Type**: `text/event-stream` → SSE handling; `application/json` (any `*/json` subtype) → buffer + parse one JSON-RPC response; anything else → pass through.

Notes and gaps to respect:

- **No JSON-RPC batch (array) handling** in practice — parsers assume a single object. Don't assume an array body parses; if you must support batch, handle it explicitly.
- **Session affinity** (`Mcp-Session-Id`) and **protocol-version negotiation** (`MCP-Protocol-Version`) are handled by Flex routing and the MCP server, **not** by gateway policies — don't invent header handling. The one exception is an `initialize`-answering policy (e.g. a router) that may upgrade a client's `protocolVersion` and echo it back.
- **stdio transport** is n/a for a gateway.

## Error Envelope

**One shape**: JSON-RPC 2.0, conventionally **HTTP 200** with the error in-band (the transport succeeded, the failure is in the body). `data` is a plain `Value` (string or structured object) — there is no A2A-style `ErrorInfo` array.

```json
{
  "jsonrpc": "2.0",
  "id": <original-id, or null>,
  "error": { "code": -32602, "message": "Invalid params", "data": "tool `foo` arguments failed schema" }
}
```

### Code conventions

| Condition | `code` |
|---|---|
| Parse error (body not JSON) | `-32700` |
| Invalid request (bad envelope) | `-32600` |
| Method not found | `-32601` |
| Invalid params (incl. schema failure) | `-32602` |
| Internal error | `-32603` |
| Access denied / unauthorized | commonly `-32007` / `-32008` (server-defined; pick what clients branch on and document it) |
| Path / upstream not found (router) | `-32000` (server-defined) |

**HTTP-status caveats** (both seen in the wild — be deliberate and consistent):

- A `JsonRpcResponse` serialised to a `Response` is naturally **HTTP 200**.
- Some shared-crate error-response builders default to **HTTP 400** for request-break errors, and accept a custom status — e.g. a PII detector that wants the client to see **HTTP 403** on rejection so it routes on status, not just code.
- `-32007`/`-32008` for auth are **server-defined conventions**, not JSON-RPC standard codes; don't assume a specific number across servers.

Always preserve the original request `id` (use `null` if it can't be recovered) and `jsonrpc:"2.0"`.

## The Request→Response Method-Threading Problem

The MCP response body has **no method field**. A response-side policy that needs to know "is this a `tools/list` result?" has two options:

1. **Thread it from the request side** — on the request filter, capture the method (and tool name for `tools/call`) into request-scoped data or a stream property, then read it back on the response filter. Preferred when you control both filters. See [[pdk-share-data-request-response]].
2. **Infer from response shape** — if `result.tools` exists it's a `tools/list`; `result.resources` → `resources/list`; `result.content` → `tools/call`; etc. Necessary on SSE streams that multiplex many responses, where a single threaded method can't describe every event.

```rust
// stream properties to carry request context to the response side
sp.set_property(&["mcp", "method"], Some(method.as_bytes()));
if method == TOOLS_CALL_METHOD_NAME {
    if let Some(name) = params.pointer("/name").and_then(Value::as_str) {
        sp.set_property(&["mcp", "tool_name"], Some(name.as_bytes()));
    }
}
```

## Pattern A — Reuse a shared crate / dispatcher (only if your workspace already has one)

If — and only if — your workspace **already** depends on a shared MCP library, reuse it. The common shape is a **typed dispatcher**: you register one handler per method, and the dispatcher parses the envelope, routes by method, validates `id` (exempting notifications), deserialises typed params only for methods that have a registered handler, and leaves everything else as `Flow::Continue`.

A request handler typically returns one of: *continue* (pass through), *respond locally* (answer the request without forwarding → `Flow::Break`), *map params* (rewrite the request body), or *error* (`Flow::Break` with a JSON-RPC error). A response handler returns *continue*, *map response*, or *error*, and a raw-map escape hatch exists for adding sibling fields (e.g. token metrics) that a `deny_unknown_fields` typed struct would reject.

Two cross-cutting gotchas the dispatcher exists to get right — **replicate them in Pattern B**:

- **Strip `Content-Length` whenever you rewrite the body** (`set_body`), or the upstream/downstream hangs until timeout (observed as a 504). Recompute or drop the header.
- **Unregistered methods pass through**; **error/empty-result responses pass through** without invoking response handlers; **upstream schema drift** (typed result won't deserialise) **passes through** rather than crashing the filter.

The control flow below (Pattern B) is exactly what a dispatcher does for you.

## Pattern B — Self-contained (default)

Ship a small per-crate `mcp.rs` (method names + content helpers) and a borrowed JSON-RPC `envelope.rs` (identical in shape to the one in [[pdk-a2a]] — MCP and A2A share the JSON-RPC envelope). ~50 LOC, no external MCP dependency.

### `mcp.rs` — method names + content helpers

```rust
pub const INITIALIZE_METHOD_NAME: &str        = "initialize";
pub const TOOLS_LIST_METHOD_NAME: &str        = "tools/list";
pub const TOOLS_CALL_METHOD_NAME: &str        = "tools/call";
pub const RESOURCES_LIST_METHOD_NAME: &str    = "resources/list";
pub const RESOURCES_READ_METHOD_NAME: &str    = "resources/read";
pub const PROMPTS_LIST_METHOD_NAME: &str      = "prompts/list";
pub const PROMPTS_GET_METHOD_NAME: &str       = "prompts/get";

pub fn is_notification(method: &str) -> bool { method.starts_with("notifications/") }

/// Methods this policy inspects on the REQUEST side (carry user/tool content).
pub fn is_request_inspectable(method: &str) -> bool {
    matches!(method, TOOLS_CALL_METHOD_NAME | RESOURCES_READ_METHOD_NAME | PROMPTS_GET_METHOD_NAME)
}

/// Methods whose RESULT carries content worth filtering/rewriting.
pub fn is_response_inspectable(method: &str) -> bool {
    matches!(method, TOOLS_LIST_METHOD_NAME | RESOURCES_LIST_METHOD_NAME
        | "resources/templates/list" | PROMPTS_LIST_METHOD_NAME | TOOLS_CALL_METHOD_NAME)
}
```

Reuse the borrowed `JsonRpcRequest` / `JsonRpcResponse` / `RpcError` / `JsonRpcId` types and `parse_jsonrpc_id` from [[pdk-a2a]]'s `jsonrpc.rs` verbatim — the envelope is the same.

### Request filter (content inspection + threading)

```rust
use pdk::hl::*;

async fn request_filter(rs: RequestState, /* pv, sp, ... */) -> Flow<()> {
    let header_state = rs.into_headers_state().await;
    let h = header_state.handler();
    if header_state.method().as_str() != "POST" { return Flow::Continue(()); }
    match h.header("content-type") {
        Some(ct) if ct.starts_with("application/json") => {}
        _ => return Flow::Continue(()),
    }

    let body_state = header_state.into_body_state().await;
    let body = body_state.as_bytes();
    let req: JsonRpcRequest<'_> = match serde_json::from_slice(body.as_slice()) {
        Ok(r) => r,
        Err(_) => return Flow::Continue(()),       // not MCP JSON-RPC → fail-open
    };

    // Notifications have no id and get no response — never error them.
    if is_notification(req.method) { return Flow::Continue(()); }

    // thread method (+ tool name) to the response side
    sp.set_property(&["mcp", "method"], Some(req.method.as_bytes()));

    if !is_request_inspectable(req.method) { return Flow::Continue(()); }
    let request_id = req.id.and_then(parse_jsonrpc_id);
    let params: Value = match req.params.and_then(|p| serde_json::from_str(p.get()).ok()) {
        Some(v) => v,
        None    => return Flow::Continue(()),
    };

    inspect_request(req.method, &params, request_id /* ... */)
}
```

### Content locations

| Method | Request pointer(s) | Response pointer(s) |
|---|---|---|
| `tools/call` | `/name`, `/arguments` | `/result/content` (each `{ type, text }`) |
| `resources/read` | `/uri` | `/result/contents` |
| `prompts/get` | `/name`, `/arguments` | `/result/messages` |
| `tools/list` | — | `/result/tools` (each `{ name, description, inputSchema }`) |
| `resources/list` | — | `/result/resources` (each `{ uri, name, ... }`) |
| `resources/templates/list` | — | `/result/resourceTemplates` |
| `prompts/list` | — | `/result/prompts` |

To **filter** a list (access control, mapping), walk the array and drop/rewrite items in place; rebuild the body and **strip `Content-Length`** before `set_body`.

### Response filter

```rust
async fn response_filter(rs: ResponseState, /* sp, ... */) -> Flow<()> {
    let header_state = rs.into_headers_state().await;
    let ct = header_state.handler().header("content-type").unwrap_or_default();
    let method = sp.read_property(&["mcp", "method"])
        .and_then(|b| String::from_utf8(b).ok())
        .unwrap_or_default();

    if ct.starts_with("text/event-stream") {
        // SSE: stream-rewrite; per-event method inference (see SSE section)
        return handle_sse(header_state /* ... */).await;
    }
    if !ct.contains("json") { return Flow::Continue(()); }      // unknown shape → pass through

    let body_state = header_state.into_body_state().await;
    let resp: JsonRpcResponse = match serde_json::from_slice(body_state.as_bytes().as_slice()) {
        Ok(r) => r,
        Err(_) => return Flow::Continue(()),                    // schema drift → pass through
    };
    if resp.error.is_some() || resp.result.is_none() { return Flow::Continue(()); }

    rewrite_result(&method, resp /* ... */)
}
```

### Error response — JSON-RPC envelope

```rust
pub fn create_mcp_error_response(id: Option<JsonRpcId>, err: RpcError, status: u32) -> Response {
    let resp = JsonRpcResponse::error(id, err);
    let body = serde_json::to_string(&resp).unwrap_or_default();
    Response::new(status)        // 200 = spec convention; a policy may use 403 etc. deliberately
        .with_headers(vec![("content-type".to_string(), "application/json".to_string())])
        .with_body(body.into_bytes())
}
```

## SSE Responses (`text/event-stream`)

Long-running MCP calls and subscriptions return SSE — each `data:` frame is one JSON-RPC response. **Never buffer SSE**; stream-rewrite. Reassemble events across chunk boundaries, parse each `data:` frame as a JSON-RPC response, rewrite if needed, re-emit. **Preserve `id:`, `event:`, and `retry:` lines** (Last-Event-ID resumability). Because one stream multiplexes many responses, use **shape inference** per event rather than the single threaded request method.

**Disable the upstream timeout** before entering body state when SSE is expected, or long-running calls get cut off (see [[pdk-request-headers-bodies]] / [[pdk-timer]]). The smallest possible MCP policy is exactly this: a request filter that only disables the timeout to enable SSE.

Use your workspace's SSE helper if it has one; otherwise the `into_sse(BodyStream)` adapter from [[pdk-a2a]] works unchanged.

## Per-Policy Patterns (what MCP policies actually do)

A quick map of the policy archetypes, so you can place a new policy and pick the right posture:

| Archetype | Inspects | Posture | Notes |
|---|---|---|---|
| **Schema validation** | all requests (envelope) + `tools/call` args vs per-tool `inputSchema` | **fail-closed** (`-32600` / `-32602`) | may serve `*/list` directly from a cached Exchange asset; background asset poll + atomic swap |
| **Access control (non-identity)** | request `tools/call`/`resources/read`/`prompts/get`; filters `*/list` response arrays | **fail-open** for non-MCP; deny → `-32008` | allow/block by literal or regex; Block wins |
| **Access control (identity-aware)** | same + caller identity (client-id contract or upstream auth) | deny-by-default; deny → `-32008` + violation | authz engine (e.g. Cedar); see [[pdk-token-introspection]], [[pdk-authentication]] |
| **Tool-name mapping** | rewrites `tools/list` response names; reverses incoming `tools/call` `params.name` | **fail-open** (never blocks) | literal + regex with capture templates; round-trip must reverse cleanly |
| **Payload optimization** | `tools/call` response `content[].text` | **fail-open** | HTML→Markdown, strip base64 blobs, collapse whitespace; skip below a min-size threshold |
| **MCP↔HTTP transcoding** | `tools/call` → outbound HTTP to a REST backend, wrap response as `CallToolResult` | error → `-32603` | per-tool path/verb/headers/body as DataWeave; answers `ping` locally; see [[pdk-http-call]], [[pdk-dataweave]] |
| **Transcoding router** | routes `tools/call`/`resources/read`/`prompts/get` to the right upstream | unknown entity → `-32602`; missing path → `-32000` | sets an upstream-selector header; answers `initialize` + `notifications/initialized` locally |
| **PII detector** | request `params` + response `result` | **fail-closed** on detection (`-32602` req / `-32603` resp, often at HTTP 403) | masks PII even inside the error envelope |
| **Telemetry / support** | classifies method; reads/strips token-metric sidecar fields on response | passive | enables SSE via no-timeout; terminal consumer of any token-metric chain |

## Local-Mode and Failure-Mode Contract

- The policy MUST load and serve from t=0 even when the control plane is unreachable.
- No `unwrap()` / `panic!` on context-derived `Option`/`Result`. Use `if let` / `match` / `unwrap_or` and log.
- Test the no-control-plane path with the PDK unit harness's local-mode builder ([[pdk-unit-tests]]).

| Condition | Posture | Why |
|---|---|---|
| Body parse fails / non-MCP traffic | **fail-open** (`Flow::Continue`), debug log | request may not be intended for an MCP server |
| Notification (no `id`) | never emit an error response | notifications are fire-and-forget |
| Unknown method in MCP envelope | reject `-32601 / METHOD_NOT_FOUND` | the envelope says MCP, the method isn't |
| Required policy config evaluation fails | **fail-closed**, `-32600` | a misconfigured policy that allows traffic is the wrong default |
| Access denied | **fail-closed**, `-32008` (+ policy violation) | by definition |
| PII detected | **fail-closed** (`-32602`/`-32603`) | PII detection failing-open is almost certainly wrong |
| Body rewritten without stripping `Content-Length` | bug → upstream 504 | always strip/recompute `Content-Length` on `set_body` |
| Upstream schema drift (typed result won't deserialise) | **pass through** | shouldn't crash the filter |
| SSE bytes already on the wire | can't recall | rewrite in-stream or skip |

## Capability Checklist

Whether you derive these (Pattern B) or get them from a shared crate (Pattern A):

| Capability | What it does |
|---|---|
| method-name constants + `is_valid_method` | name the vocabulary; reject unknown methods |
| `is_notification` | detect `notifications/*` (no id, no error response) |
| request/response inspectable predicates | which methods carry content on which side |
| borrowed JSON-RPC parser + error envelope | shared with A2A; HTTP-200 convention |
| request→response threading (or shape inference) | carry method/tool-name to the response side |
| content-location pointers | request `params` and response `result` walks |
| body-rewrite with `Content-Length` strip | mutate request or response without a 504 |
| error-response builder (status-parameterised) | 200 by default; 403 etc. when deliberate |
| SSE stream-rewrite + timeout disable | per-event parse/rewrite, preserve `id:`/`event:`/`retry:` |
| token-metric sidecar (optional) | add `_*` fields via raw-map; strip them at the telemetry consumer |

## Common Mistakes

| Mistake | Fix |
|---|---|
| Returning HTTP 4xx/5xx for a JSON-RPC failure | use **HTTP 200** with the error in-band (a deliberate 403 for client routing is the documented exception) |
| Emitting an error response to a notification | notifications have no `id` — never respond |
| Assuming JSON-RPC batch arrays parse | parsers are single-object; handle batch explicitly if needed |
| Inventing `Mcp-Session-Id` / `MCP-Protocol-Version` handling | session/version is Flex routing + the server, not policy |
| Hard-coding `"tools/call"` literals | declare `pub const` once (or reuse a shared crate's constants) |
| Rewriting body without stripping `Content-Length` | strip/recompute it on `set_body` or the upstream hangs (504) |
| Carrying extra fields on a `deny_unknown_fields` typed response | use the raw-map escape hatch (mutate bytes, not the typed struct) |
| Buffering an SSE response | stream-rewrite; preserve `id:`/`event:`/`retry:` |
| Threading one method onto a multiplexed SSE stream | infer the method per event from response shape |
| Filtering a `*/list` on the request side | the list is in the **response** — filter there |
| Calling `unwrap()` on a context-derived `Option` | local-mode must not panic; `if let` / `match` / `unwrap_or` + log |
| Adding a shared MCP crate just to get the constants | derive Pattern B inline; only adopt a crate already in the workspace |

## Red Flags

- "I'll buffer the SSE response so I can rewrite it" — no; stream-rewrite or skip.
- "I'll return HTTP 400 for the JSON-RPC parse failure" — JSON-RPC errors are in-band at HTTP 200 (deliberate 403 aside).
- "I'll send an error back for this `notifications/cancelled`" — notifications get no response, ever.
- "I'll read the method off the response body" — there is no method on the response; thread it or infer from shape.
- "I'll set the new body and move on" — strip `Content-Length` first or you'll get a 504.
- "I'll add a shared MCP crate dependency to get the dispatcher" — derive Pattern B inline unless the crate is **already** a workspace dependency.
- "I'll match `request.method` against literals" — declare `pub const` once or reuse existing constants.

## Cross-References

- [[pdk-a2a]] — sibling skill; same JSON-RPC envelope, same SSE patterns, same portability principle. Reuse its `jsonrpc.rs`.
- [[pdk-create-policy]] — split-model crate layout for new MCP policies.
- [[pdk-json-validator]] / [[pdk-schema-definition]] — schema validation of `tools/call` arguments and `gcl.yaml` config.
- [[pdk-authentication]] / [[pdk-token-introspection]] / [[pdk-contracts-validation]] — caller identity for access-control policies.
- [[pdk-policy-violations]] — required when blocking on a deny decision (access denied, PII detected, schema invalid).
- [[pdk-http-call]] / [[pdk-dataweave]] — MCP↔HTTP transcoding (outbound call + per-tool DataWeave mapping).
- [[pdk-request-headers-bodies]] — reading/rewriting bodies, controlling the upstream timeout for SSE.
- [[pdk-stop-execution]] — short-circuiting with `Flow::Break(response)`.
- [[pdk-share-data-request-response]] — threading the MCP method / tool name from request to response filter.
- [[pdk-rate-limiting]] — per-tool / per-caller rate limiting of MCP calls.
- [[pdk-embedding-services]] / [[pdk-vector-stores]] — semantic MCP policies, parallel to A2A.
- [[pdk-unit-tests]] / [[pdk-integration-tests]] — unit + integration recipes (SSE responses, JSON-RPC envelope assertions, local-mode).
