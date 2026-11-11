---
name: pdk-sse-parsing
description: "Use when a PDK custom policy must parse Server-Sent Events (text/event-stream) in a streaming response body - LLM transcoding, streaming-API inspection or rewriting. Covers blank-line event boundaries, multi-line data fields, optional spaces, the data:[DONE] sentinel, the buffer-drain loop, fail-closed inspecting rewrites (PII masking or blocking mid-stream), and reading usage from stream properties after draining."
---

# Skill: Parsing Server-Sent Events (SSE)

## Topic: Implementation

Parse `text/event-stream` response bodies inside a PDK policy — typically an LLM transcoding
policy converting one streaming format to another, or a policy that inspects/rewrites streamed
events. SSE is a **streaming** MIME type: buffering the whole body breaks the streaming contract
([[pdk-runtime-model]]), so parse incrementally.

## SSE wire format

Text-based, one event per block. Each event is a run of `field: value` lines terminated by a
**blank line** (`\n\n`, or `\r\n\r\n` for CRLF). Relevant fields:

- `event: <type>` — optional event type
- `data: <payload>` — payload; may repeat within one event
- `id: <id>` — optional

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01"}}

```

## Parsing rules that bite

1. **The boundary is the blank line** — accept both `\n\n` and `\r\n\r\n`. A parser that only
   handles LF silently stalls on a CRLF stream.
2. **Multiple `data:` lines in one event concatenate with `\n`** (per spec) — collect them all,
   then parse the joined payload as one unit.
3. **The space after `data:` is optional** — `data: x` and `data:x` are both valid; strip a
   leading space if present, don't require it.
4. **`event:` may be absent** — some producers send only `data:` lines.
5. **Sentinels are not JSON** — OpenAI ends a stream with the literal `data: [DONE]`. Deserializing
   fails on it; treat a parse failure as skip-and-continue (`debug!`), never `unwrap`.

## Buffer-drain loop over a streaming body

Accumulate chunks in a `Vec<u8>`, extract every complete event the buffer now holds, drain the
consumed bytes, repeat until the stream ends.

```rust
use pdk::logger;

async fn handle_sse_response(headers: ResponseHeadersState) {
    let body_stream_state = headers.into_body_stream_state().await;
    let mut stream = body_stream_state.stream();
    let mut buf: Vec<u8> = Vec::new();

    while let Some(chunk) = stream.next().await {
        buf.extend_from_slice(chunk.bytes());

        // Drain every complete event currently buffered; a partial tail stays for the next chunk.
        while let Some((data_payload, consumed)) = take_one_sse_event(&buf) {
            match serde_json::from_str::<MyEvent>(&data_payload) {
                Ok(event) => { /* map / inspect / rewrite */ }
                Err(e) => logger::debug!("skipping non-JSON SSE payload: {e}"), // e.g. [DONE]
            }
            buf.drain(..consumed);
        }
    }
}

/// Extract one complete event from `buf`: the concatenated `data:` payload and the number of
/// bytes consumed (payload + boundary). `None` until a full event is buffered.
fn take_one_sse_event(buf: &[u8]) -> Option<(String, usize)> {
    let (boundary_start, boundary_len) = find_event_boundary(buf)?;
    let event_str = std::str::from_utf8(&buf[..boundary_start]).ok()?;

    let mut data_lines: Vec<&str> = Vec::new();
    for line in event_str.split('\n') {
        let line = line.trim_end_matches('\r');
        if let Some(payload) = line.strip_prefix("data:") {
            data_lines.push(payload.strip_prefix(' ').unwrap_or(payload)); // optional space
        }
    }
    Some((data_lines.join("\n"), boundary_start + boundary_len))
}

/// First `\n\n` or `\r\n\r\n`. Returns `(index, boundary_len)`.
fn find_event_boundary(buf: &[u8]) -> Option<(usize, usize)> {
    let mut i = 0;
    while i + 1 < buf.len() {
        if buf[i] == b'\n' && buf[i + 1] == b'\n' {
            return Some((i, 2));
        }
        if i + 3 < buf.len()
            && buf[i] == b'\r' && buf[i + 1] == b'\n'
            && buf[i + 2] == b'\r' && buf[i + 3] == b'\n'
        {
            return Some((i, 4));
        }
        i += 1;
    }
    None
}
```

## Test the boundaries

Unit-test `take_one_sse_event` directly ([[pdk-unit-tests]]): LF boundary, CRLF boundary, partial
event returns `None`, event with no `event:` line, `data:` with no space, multiple `data:` lines
joined, and non-`data:` lines skipped.

## Rewriting events back into the stream

This skill covers the **read/parse** side. Emitting rewritten events back downstream is a separate,
constrained problem:

- **Never buffer the whole SSE body to rewrite it.** That breaks the streaming contract the client
  depends on ([[pdk-runtime-model]]) and is capped by the **per-connection** Envoy connection buffer
  (`FLEX_DOWNSTREAM_CONNECTION_BUFFER_LIMIT_BYTES`, "Global connection buffer limit" in the managed
  Flex Gateway UI) — a long-lived stream will exceed it.
- **Per-chunk writing is experimental-only, and not an escape hatch.** The default stream body state
  is read-only; `write_chunk` exists only behind the aggregate `experimental` Cargo feature, is not
  GA, and each write is still checked against the body-size guard **and remains bounded by the same
  per-connection connection buffer named above** — enabling it does not let a long-lived stream write
  past that cap. See [[pdk-request-headers-bodies]] and [[pdk-experimental-feature]].
- **Rewrite event-by-event, not payload-by-payload.** You can only decide on complete events as they
  drain from the buffer above, so keep transformations local to a single event; anything that needs
  the whole stream to decide its output cannot be done as a streaming rewrite.

### Inspecting rewrite (masking / blocking): fail closed, never await between chunks

A policy that must *inspect* every event before it reaches the client (PII masking, content
blocking) has a stronger contract than a cosmetic rewrite. The shape that works:

```rust
async fn handle_sse_response(headers_state: ResponseHeadersState, violations: &PolicyViolations) {
    // The final length is unknown, so drop Content-Length and let Envoy stream chunked.
    headers_state.handler().remove_header(CONTENT_LENGTH_HEADER);
    let body = headers_state.into_body_stream_state().await;  // write_chunk: `experimental` feature
    let mut stream = body.stream();
    let mut buf = Vec::new();
    let mut poisoned = false;

    while let Some(chunk) = stream.next().await {
        buf.extend_from_slice(chunk.bytes());
        let mut out = Vec::new();
        while let Some((event, consumed)) = take_one_raw_event(&buf) {  // keeps id:/event: lines
            if !poisoned {
                out.extend(inspect_event(event, &mut poisoned, violations)); // SYNC, no .await
            }
            buf.drain(..consumed);
        }
        if body.write_chunk(Chunk::new(out)).is_err() { return; } // client disconnected
    }
}
```

- **Never `.await` between reading a chunk and writing its output** (no outbound HTTP call per
  event). While the filter is suspended, the body buffer keeps accumulating upstream bytes, so whole
  events get lost, duplicated or garbled, and output lags by a chunk. Keep per-event inspection
  synchronous. If a decision really needs a remote call, buffer and moderate the full response
  instead (see the guardrail section in [[pdk-http-call]]).
- **Fail closed with a poison flag.** If an event's payload can't be inspected (larger than your
  inspection cap, or not valid UTF-8), set `poisoned = true`, emit one replacement event, and drop
  every later event. Never forward uninspected bytes, and never decode lossily to "get past" a bad
  payload.
- **Report the failure in-stream.** The HTTP status and headers are already on the wire, so you
  can't send a 403 anymore. Emit a protocol-level error event instead. For JSON-RPC (MCP, A2A):
  `data: {"jsonrpc":"2.0","id":null,"error":{"code":-32603,"message":"Internal error","data":{"reason":"…"}}}`,
  and also raise a policy violation ([[pdk-policy-violations]]).
- Write one output chunk per input chunk, even an empty one, so backpressure follows the upstream.
  Handle a leading UTF-8 BOM and lone-CR line endings as well as `\n` and `\r\n`.

### Drain, then act on what other policies published

A response-side policy that only needs a **summary** of the stream (for example token usage for
rate limiting or billing) shouldn't parse it if another policy already extracted that data. Drain
the stream without touching the bytes, and only then read the stream properties
([[pdk-share-data-between-policies]]):

```rust
let body = headers_state.into_body_stream_state().await;
let mut stream = body.stream();
while stream.next().await.is_some() {}           // pass-through: bytes flow to the client untouched

let read_u64 = |path: &[&str]| {
    stream_props.read_property(path)
        .and_then(|b| String::from_utf8(b).ok())
        .and_then(|s| s.trim().parse::<u64>().ok())   // published as decimal text
};
if let Some(total) = read_u64(&["myorg.llm.tokens.total"]) {
    charge(total);
} // else: the producer didn't run or couldn't extract usage, so decide explicitly (skip or estimate)
```

Properties written by a policy earlier in the chain are complete only **after** the stream ends,
so read them after the drain loop, not at response-headers time.

## When NOT to use

- **Binary streaming** (gRPC, WebSocket binary frames, raw chunked transfer) — different framing;
  see [[pdk-websockets]].
- **A policy that streams a body through without reading events** — use `BodyStream` directly
  ([[pdk-request-headers-bodies]]); no event extraction needed.

## Source Ref

- Generalized PDK pattern (SSE per the WHATWG/HTML `text/event-stream` spec). Not covered by the
  PDK docs as of this writing — treat the parsing rules above as the contract.
- The fail-closed inspecting rewrite and the drain-then-read-properties idiom (2026-11-11) are
  field-derived from MuleSoft's own MCP and LLM policies. `write_chunk` needs the `experimental`
  Cargo feature in PDK 1.10.0.
