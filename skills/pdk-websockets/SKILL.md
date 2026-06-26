---
name: pdk-websockets
description: Use when implementing PDK policies for WebSocket APIs using pdk::websockets library (1.9.0 open beta), covering FilterBuilder upgrade hooks (on_create, on_upgrade_upstream, on_upgrade_downstream, on_done), frame boundary accumulation with Decoder/Encoder, client/server bidirectional frame processing, and pdk-unit WebSocket upgrade testing with UnitFrame and UpgradeConnection.
---

# Skill: Support WebSocket APIs

## Topic: Implementation

This skill covers how to write PDK custom policies for WebSocket APIs using the PDK WebSocket
library (`pdk::websockets`). Introduced in **PDK 1.9.0** as an **open beta**, gated behind the
`experimental_websocket` feature.

## Enable WebSocket Support

Add the `experimental_websocket` feature to the `pdk` dependency in `Cargo.toml`. Frame-level
manipulation also needs the low-level `ll` feature:

```toml
[dependencies]
pdk = { version = "1.9.0", features = ["ll", "experimental_websocket"] }
```

## Configure WebSocket Handlers

A WebSocket policy runs as an upgrade filter. Use `FilterBuilder` with `on_upgrade_upstream`
(client → server) and `on_upgrade_downstream` (server → client). `on_create` initializes
per-connection state; `on_done` runs cleanup/logging when the session ends.

```rust
mod generated;

use anyhow::Result;
use pdk::hl::*;
use pdk::websockets::{Decoder, Encoder, Frame, FrameType};

type BoxError = Box<dyn std::error::Error>;

#[derive(Clone)]
struct CounterState {
    frame_count: u64,
}

#[pdk::entrypoint]
pub async fn configure(launcher: Launcher) -> Result<()> {
    let handler = FilterBuilder::new()
        .on_create(|| CounterState { frame_count: 0 })
        .on_request(|_req: RequestState| async move { Flow::Continue(()) })
        .on_upgrade_upstream(|state: UpstreamState, State(st): State<CounterState>| {
            handle_upstream(state, st.frame_count)
        })
        .on_upgrade_downstream(|state: DownstreamState, _: State<CounterState>| {
            handle_downstream(state)
        })
        .on_done(|st: CounterState| {
            pdk::logger::info!("WebSocket session closed — {} frames", st.frame_count);
        })
        .build();

    launcher.launch(handler).await?;
    Ok(())
}
```

## Supported Frame Types

`FrameType` variants: `Text`, `Binary`, `Close`, `Ping`, `Pong`.

**Pass control frames (`Close`, `Ping`, `Pong`) through unmodified** to preserve WebSocket protocol
integrity. Only transform `Text` / `Binary` data frames.

## Maintain Frame Boundaries

WebSocket frames may be fragmented across multiple network chunks. Your handler must process
**complete** messages regardless of arrival pattern:

1. Accumulate bytes with `state.accumulate().await` when no complete frame is available yet.
2. Track remainder (leftover partial-frame) bytes between loop iterations.
3. Parse complete frames only once sufficient data has arrived.

## Process Client → Server Frames (Upstream)

Use `UpstreamState`. `Encoder::encode_client` re-encodes frames heading to the server.

```rust
async fn handle_upstream(mut state: UpstreamState, mut frame_count: u64) -> Result<(), BoxError> {
    let mut remainder = Vec::new();

    loop {
        let mut bytes = remainder.clone();
        bytes.extend_from_slice(&state.bytes());

        let (frames, leftover) = Decoder::parse(bytes);

        if frames.is_empty() {
            // Incomplete frame — wait for more data
            state = state.accumulate().await;
        } else {
            remainder = leftover;

            let transformed: Vec<Frame> = frames
                .into_iter()
                .map(|frame| match frame.frame_type() {
                    FrameType::Text => {
                        frame_count += 1;
                        let original = String::from_utf8_lossy(frame.data());
                        Frame::text(format!("{frame_count}:{original}"), frame.fin())
                    }
                    _ => frame, // Binary + control frames pass through
                })
                .collect();

            let encoded = Encoder::default().encode_client(transformed);
            state.set_body(&encoded);
            state = state.next().await;
        }
    }
}
```

## Process Server → Client Frames (Downstream)

Use `DownstreamState`. `Encoder::encode_server` re-encodes frames heading to the client. Same
accumulate/parse/transform/encode loop as upstream.

```rust
async fn handle_downstream(mut state: DownstreamState) -> Result<(), BoxError> {
    let mut remainder = Vec::new();

    loop {
        let mut bytes = remainder.clone();
        bytes.extend_from_slice(&state.bytes());

        let (frames, leftover) = Decoder::parse(bytes);

        if frames.is_empty() {
            state = state.accumulate().await;
        } else {
            remainder = leftover;

            let transformed: Vec<Frame> = frames
                .into_iter()
                .map(|frame| match frame.frame_type() {
                    FrameType::Text => {
                        let text = String::from_utf8_lossy(frame.data());
                        Frame::text(format!("Echo: {text}"), frame.fin())
                    }
                    _ => frame,
                })
                .collect();

            let encoded = Encoder::default().encode_server(transformed);
            state.set_body(&encoded);
            state = state.next().await;
        }
    }
}
```

## Frame API

- `Decoder::parse(bytes) -> (Vec<Frame>, Vec<u8>)` — returns complete frames + leftover remainder.
- `Frame::text(data, fin)`, `Frame::binary(data, fin)` — construct data frames; `fin` marks the
  final fragment of a message.
- `frame.frame_type()`, `frame.data()`, `frame.fin()` — accessors.
- `Encoder::default().encode_client(frames)` / `.encode_server(frames)` — re-encode for the
  respective direction.

## Testing WebSocket Policies (pdk-unit)

`pdk-unit` (1.9.0+) supports WebSocket upgrade tests. Configure the backend with
`UnitHttpResponse::upgrade()`, then call `upgrade(UnitHttpRequest::upgrade())` instead of `request`.
The returned `UpgradeConnection` exposes `response()`, `client()`, and `server()`.

```rust
use pdk_unit::{
    UnitFrame, UnitFrameType, UnitHttpRequest, UnitHttpResponse, UnitTest, UnitTestBuilder,
};

fn tester() -> UnitTest {
    UnitTestBuilder::default()
        .with_config("{}")
        .with_backend(UnitHttpResponse::upgrade())
        .with_entrypoint(configure)
}

#[test]
fn text_frames_are_prefixed_with_counter() {
    let mut t = tester();
    let conn = t.upgrade(UnitHttpRequest::upgrade()).unwrap();

    conn.client().send_to_server(UnitFrame::text("hello", true));
    let frame = conn.server().next().unwrap();
    assert_eq!(String::from_utf8_lossy(frame.data()), "1:hello");
}
```

- `UnitFrame::text(payload, fin)`, `::binary(payload, fin)`, `::ping()`, `::pong()`, `::close()`.
- `UnitFrameType`: `Text`, `Binary`, `Ping`, `Pong`, `Close`.
- `conn.client().send_to_server(frame)` / `conn.server().send_to_client(frame)`.
- `conn.server().next()` / `conn.client().next()` — pull the next forwarded frame.
- `t.set_chunk_size(n)` — force TCP fragmentation to test partial-frame accumulation. (Default
  chunk size is 30KB as of 1.9.0; was 3 bytes pre-1.9.0.)

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-websocket
- pdk-unit WebSocket testing: https://docs.mulesoft.com/pdk/latest/policies-pdk-unit#test-websocket-policies
- Example: https://github.com/mulesoft/pdk-custom-policy-examples/tree/1.9.0/websocket

## Related Skills

- [[pdk-unit-tests]] — full pdk-unit testing reference
- [[pdk-request-headers-bodies]] — HTTP body manipulation (non-WebSocket)
- [[pdk-create-policy]] — scaffold a new policy project
