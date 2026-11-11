---
name: pdk-telemetry-events-spans
description: Use when a PDK policy must emit structured telemetry events (OTLP log records such as GenAI prompt/completion events) through an HttpEventExporter, or name and tag the upstream tracing span with UpstreamSpan (pdk::metrics::event, pdk::metrics::span), including gen_ai.* span tags leaking onto the root/server span, events lost on client abort, and unit-testing events via OTLP JSON.
---

# Skill: Telemetry Events and Upstream Spans

## Topic: Implementation (internal / unreleased PDK surface)

> **Caveat: not public API.** `pdk::metrics::event` (`Event`, `KeyValue`, `Severity`, `Value`,
> `HttpEventExporter`) and `pdk::metrics::span::UpstreamSpan` are **not in public PDK 1.10.0 or
> 1.10.1-alpha.0** (no docs page, absent from docs.rs). MuleSoft's own GenAI policies use them from
> internal PDK builds. Signatures may change or never ship. A policy built against public PDK
> 1.10.0 won't compile with these imports. Use them only when your target PDK actually exposes
> them, and keep the policy working when they're absent. For public alternatives, see
> [[pdk-policy-logging]] (logs) and [[pdk-share-data-between-policies]] (values for the tracing
> policy's `requestHeader` labels).

## Inject the exporter as `Option<HttpEventExporter>`

The exporter is configured by the gateway, not by your policy. Inject it as an `Option` in
`#[entrypoint]`. It is `None` when no event destination is configured (local mode, invalid config).
That's not an error, so fall back to logging and keep serving traffic:

```rust
use pdk::metrics::event::HttpEventExporter;

#[entrypoint]
async fn configure(
    launcher: Launcher,
    configuration: Configuration,
    event_exporter: Option<HttpEventExporter>,
) -> Result<()> {
    if event_exporter.is_none() {
        logger::warn!("[my-policy] no event exporter; events are logged at debug only");
    }
    let exporter = event_exporter.as_ref();
    launcher
        .launch(on_request(|rs| request_filter(rs)).on_response(|rs, sp| response_filter(rs, sp, exporter)))
        .await?;
    Ok(())
}
```

## Build an `Event`

`Event` is an OTLP log record with a builder API:

```rust
use pdk::metrics::event::{Event, KeyValue, Severity, Value};

let mut event = Event::new("gen_ai.client.inference.operation.details") // event name
    .body(&body_json)                    // string body
    .timestamp_unix_nanos(now_ns)
    .severity(Severity::Info)            // INFO (9) is the default
    .attribute(KeyValue::new("gen_ai.response.model", Value::from(model.as_str())))
    .attribute(KeyValue::new("gen_ai.usage.input_tokens", Value::from(input_tokens as i64)));

// Correlate with the active trace only when the ids parse. A malformed id is LEFT OUT, and the
// event is still sent.
if let Some(trace_id) = parse_trace_id(&traceparent) { event = event.trace_id(trace_id); } // [u8; 16]
if let Some(span_id) = parse_span_id(&traceparent)   { event = event.span_id(span_id); }   // [u8; 8]
```

`Value` converts from `&str`, `i64`, `f64` and `bool`, and holds an array of strings for list
attributes (for example `gen_ai.response.finish_reasons`).

## Queue per request, flush once, fail open

Don't send per event, and don't send mid-stream. Collect the events for the request in a
single-threaded queue, then send them in **one** call at the end of the response filter, after the
body has been fully read and released. By then the last chunk is already on its way to the client,
so the POST adds no client-visible latency:

```rust
#[derive(Default)]
struct EventQueue { queue: RefCell<Vec<Event>> }   // RefCell, not Mutex: proxy-wasm is single-threaded

impl EventQueue {
    fn push(&self, e: Event) { self.queue.borrow_mut().push(e); }
    fn drain(&self) -> Vec<Event> { self.queue.borrow_mut().drain(..).collect() }
}

impl Drop for EventQueue {
    // An async send can't run in Drop. If the filter is dropped before the flush (client abort),
    // record the loss instead of losing it silently.
    fn drop(&mut self) {
        let left = self.queue.get_mut().len();
        if left > 0 { logger::warn!("[my-policy] {left} telemetry event(s) not exported (request aborted)"); }
    }
}

async fn flush_events(exporter: &HttpEventExporter, queue: &EventQueue) {
    let events = queue.drain();
    if events.is_empty() { return; }
    if let Err(err) = exporter.send(&events).await {
        // Telemetry is fail-open: log once, drop, no retry, never affects the response.
        logger::warn!("[my-policy] failed to export {} event(s), dropped: {err}", events.len());
    }
}
```

- Build the queue only when there's an exporter **and** something to report. Otherwise write
  straight to the logging fallback.
- Never block or reject a request because telemetry failed.
- Events are as sensitive as logs. Prompt or completion bodies belong in events only when the
  operator has opted in, and never include credentials ([[pdk-policy-logging]]).

## Name and tag the upstream span: `UpstreamSpan`

`UpstreamSpan` writes the name and tags of the **upstream (CLIENT/egress)** span through stream
properties. Use it in a filter that runs on the outbound side, before the egress span is finalized:

```rust
use pdk::metrics::span::UpstreamSpan;

let span = UpstreamSpan::new(&stream_properties);
span.set_name(&format!("chat {model}"))
    .set_tag("gen_ai.operation.name", "chat")
    .set_tag("gen_ai.response.model", &model);

// Later, once the response is known:
UpstreamSpan::new(&stream_properties).set_tag("gen_ai.usage.input_tokens", &input.to_string());
```

Common GenAI semantic-convention tags are `gen_ai.provider.name`, `gen_ai.request.model`,
`gen_ai.response.model`, `gen_ai.usage.{input,output,total}_tokens`,
`gen_ai.usage.cache_read.input_tokens`, `gen_ai.usage.cache_write.input_tokens`,
`gen_ai.usage.reasoning.output_tokens` and `gen_ai.response.finish_reasons`. Tag values must be
bounded and low-cardinality, never raw prompts ([[pdk-coding-best-practices]]).

### Clear the tags so they don't leak onto the root span

The tags share stream properties that the **downstream (SERVER/root)** span also reads. If you
leave them in place, every `gen_ai.*` tag is stamped on both spans. In a policy on the
**inbound** side, drain the response stream and then call `clear_tags()`:

```rust
let body = headers_state.into_body_stream_state().await;
let mut stream = body.stream();
while stream.next().await.is_some() {}             // pass-through; wait for the stream to end
UpstreamSpan::new(&stream_properties).clear_tags(); // empties registered tag values, keeps the name
```

Timing matters. After the drain, the CLIENT span has already captured the tags, and the SERVER span
hasn't been finalized yet. So clearing at that point removes the tags from the SERVER span only.
Clearing earlier, for example at response-headers time, removes them from both spans.

## Testing

`event.to_otlp_json()` renders the OTLP JSON for an event. Assert on it in unit tests
([[pdk-unit-tests]]): the name, the default severity (`INFO`/9), the attributes, and that a
malformed trace id is left out instead of failing. In unit tests, use a fake exporter or skip the
flush entirely.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Injecting `HttpEventExporter` (not `Option`) | Inject `Option<HttpEventExporter>`; `None` is a normal configuration |
| Sending one POST per event, or sending mid-stream | Queue, then flush once after the body is released |
| Failing the request when export fails | Warn once and drop |
| Using `Mutex`/`Arc` for the queue | `RefCell` (single-threaded runtime) |
| `gen_ai.*` tags appear on the root span too | `clear_tags()` on the inbound side **after** the stream drains |
| Dropping an event because its trace id is malformed | Leave out the id, keep the event |
| Assuming the API exists in public PDK 1.10.0 | It doesn't; gate on the PDK you actually build against |

## Source Ref

- **Status:** internal/unreleased PDK API. Checked 2026-11-11: no page on docs.mulesoft.com, and
  `pdk::metrics::event` / `pdk::metrics::span` are absent from https://docs.rs/pdk/1.10.0 and
  1.10.1-alpha.0.
- **Derived from:** field observation of MuleSoft's own GenAI gateway policies (2026-11-11). The
  `gen_ai.*` names follow the OpenTelemetry GenAI semantic conventions:
  https://opentelemetry.io/docs/specs/semconv/gen-ai/
