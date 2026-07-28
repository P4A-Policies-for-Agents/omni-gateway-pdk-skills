---
name: pdk-sse-parsing
description: Use when a PDK custom policy must parse Server-Sent Events (text/event-stream) in a streaming response body — LLM transcoding, streaming-API inspection or rewriting. Covers the SSE event-boundary rule (blank line, \n\n and \r\n\r\n), multi-line data: concatenation, optional space after the colon, the data:[DONE] sentinel, and the buffer-drain loop over a streaming body.
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

## When NOT to use

- **Binary streaming** (gRPC, WebSocket binary frames, raw chunked transfer) — different framing;
  see [[pdk-websockets]].
- **A policy that streams a body through without reading events** — use `BodyStream` directly
  ([[pdk-request-headers-bodies]]); no event extraction needed.

## Source Ref

- Generalized PDK pattern (SSE per the WHATWG/HTML `text/event-stream` spec). Not covered by the
  PDK docs as of this writing — treat the parsing rules above as the contract.
