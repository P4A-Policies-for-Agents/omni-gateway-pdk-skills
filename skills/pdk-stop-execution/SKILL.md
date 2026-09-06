---
name: pdk-stop-execution
description: Use when sending early responses to terminate execution flow in PDK policies, using Flow::Break(response) in request filters to abort requests or ResponseHeadersState.send_response(response) to terminate response processing with custom status, headers, and body.
---

# Skill: Stopping Request and Response Execution

## Topic: Implementation

This skill covers how to send early responses during request and response execution to terminate the execution flow.

## Response Object

Build a response with status code, headers, and body:

```rust
Response::new(401)
    .with_headers(vec![("WWW-Authenticate".to_string(), "Bearer realm=\"oauth2\"".to_string())])
    .with_body(r#"{ "error": "token was not present"}"#)
```

## Stop Request Execution

Use `Flow` enum to block or allow requests:
- `Flow::Continue(data)`: Forwards data to the response
- `Flow::Break(response)`: Aborts request and returns the provided response

```rust
async fn request_filter(request_state: RequestState) -> Flow<()> {
    let header_state = request_state.into_headers_state().await;
    let handler = header_state.handler();
    if handler.header("authorization").is_some() {
        Flow::Continue(())
    } else {
        Flow::Break(Response::new(401)
            .with_headers(vec![("WWW-Authenticate".to_string(), "Bearer realm=\"oauth2\"".to_string())])
            .with_body(r#"{ "error": "token was not present"}"#))
    }
}
```

**Important:** Due to streaming nature of Proxy Wasm, an early request may partially reach the upstream while awaiting `headers_state.into_body_state`. Avoid by not awaiting the body, or configure upstream to ignore partial requests.

### Terminating policies must buffer headers+body atomically

A policy that **answers the request itself** — a bridge, a well-known endpoint, a rejection that
needs the body, any synthetic response — must not use the sequential
`into_headers_state().await` → `into_body_state().await` pattern. The first `await` releases the
headers to Envoy's router, which begins proxying the request upstream **in parallel** while your
policy runs. If you then `Flow::Break`, the upstream response (e.g. a 404) can race and beat your
synthetic one.

Instead, buffer both in one transition with `into_headers_body_state().await` (requires the
`enable_stop_iteration` feature). Envoy holds the request in the filter and never forwards it, so
your `Flow::Break` always wins. Envoy buffers the body in the **per-connection** connection buffer,
sized by `FLEX_DOWNSTREAM_CONNECTION_BUFFER_LIMIT_BYTES` (default ~1 MB; in managed Flex Gateway the
UI field is **"Global connection buffer limit"**). There is no hard configuration ceiling — the max
is bound mainly by the host's available memory (10 MB is a valid value). Size it to the largest
accepted body. Note this Envoy connection buffer is a separate mechanism from the PDK **< 1.10**
hardcoded 1 MB `set_body` write-check. PDK 1.10+ with the corresponding Omni fix reads the configured
limit and **fails an oversized write gracefully** (no panic), but the limit remains physical and
finite; on PDK < 1.10 an experimental body-limit bypass can instead **panic on an oversized response
body or return a 413 on an oversized request body** (see [[pdk-experimental-feature]]). See
[[pdk-request-headers-bodies]].

```rust
// Terminating policy: atomic buffer, then answer. Envoy never forwards upstream.
let state = request_state.into_headers_body_state().await;
let method = state.handler().header(":method").unwrap_or_default();
// ... inspect headers+body, build the synthetic response ...
Flow::Break(Response::new(200).with_body("answered by this policy"))
```

This does **not** apply to pass-through policies (validate/decorate/transform-then-continue) —
those correctly forward headers before the body arrives.

For fail-closed validation, streaming is not automatically equivalent to atomic buffering: the
separate event flow can forward headers or partial body data while validation is still running.
Reject known-oversized requests from headers when possible, enforce a bounded accepted size, or
move verification/buffering outside the policy when arbitrary-size input must be held before any
upstream forwarding.

## Stop Response Execution

Use `send_response` method on `ResponseHeadersState`. Cannot send early response from body state. Terminates execution flow — no further operations possible:

```rust
pub fn send_response(self, response: Response);

async fn response_filter(state: ResponseState) {
   let state = state.into_headers_state().await;
   state.send_response(
       Response::new(200)
           .with_headers(vec![("some".to_string(), "200".to_string())])
           .with_body("Some"),
   );
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-stop

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-stop.adoc`
- **Snapshot:** 2026-08-24
