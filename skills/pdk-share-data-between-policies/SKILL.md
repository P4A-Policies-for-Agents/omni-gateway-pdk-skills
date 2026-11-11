---
name: pdk-share-data-between-policies
description: Use when sharing properties between multiple policies processing the same request using the StreamProperties injectable and PropertyAccessor trait (read_property, set_property) to broadcast and consume data across policy instances, or when reading built-in gateway/Envoy connection attributes (mTLS peer certificate subject, client source address, request id) that the gateway populates on the stream, or when publishing a computed value (token usage, model, identity) once for downstream policies, rate limiting, metrics or tracing labels to consume.
---

# Skill: Sharing Data Between Policies

## Topic: Implementation

This skill covers how to share properties between policies processing the same request using `StreamProperties`.

## Overview

The `StreamProperties` injectable provides an interface to:
- Consume properties set by other policies processing the same request
- Broadcast properties for other policies to consume

## PropertyAccessor Trait

```rust
pub trait PropertyAccessor {
    fn read_property(&self, path: &[&str]) -> Option<Bytes>;
    fn set_property(&self, path: &[&str], value: Option<&[u8]>);
}
```

## Example

```rust
async fn request_filter(stream: StreamProperties) -> Flow<()> {
    let incoming = String::from_utf8(
        stream.read_property(&["incoming_property"]).unwrap_or_default()
    ).unwrap_or_default();

    logger::info!("Received incoming prop {}", incoming);

    let outgoing = "outgoing".as_bytes();
    stream.set_property(&["outgoing_property"], Some(outgoing));

    Flow::Continue(())
}

#[entrypoint]
async fn configure(launcher: Launcher) -> Result<()> {
    let filter = on_request(|stream| request_filter(stream));
    launcher.launch(filter).await?;
    Ok(())
}
```

## Built-in gateway/Envoy attributes (not just custom properties)

`read_property` reads two kinds of properties through the same API:

1. **Custom properties** another policy set with `set_property` (the examples above).
2. **Built-in attributes the gateway/Envoy populate** on every stream — these are *read-only*
   and exist even when no other policy ran. Address them by their well-known path segments:

```rust
async fn request_filter(request_state: RequestState, stream: StreamProperties) -> Flow<()> {
    let headers_state = request_state.into_headers_state().await;

    // mTLS peer certificate subject (DN string), e.g. "CN=Alice,emailAddress=alice@example.com"
    let peer_subject = String::from_utf8_lossy(
        &stream.read_property(&["connection", "subject_peer_certificate"]).unwrap_or_default()
    ).to_string();

    // Other commonly-used built-ins:
    // &["source", "address"]  -> downstream client address (client IP:port)
    // &["request", "id"]      -> the request id (also injected into log lines)

    headers_state.handler().set_header("x-peer-subject", peer_subject.as_str());
    Flow::Continue(())
}
```

The full attribute set is Envoy's, not PDK's — see the Envoy attribute reference:
- Connection/TLS attributes: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#connection-attributes
- Upstream attributes: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#upstream-attribute

A missing/unset attribute returns `None`; `.unwrap_or_default()` yields empty bytes, so a policy that
reads the peer cert must handle the no-mTLS case rather than assuming the value is present.

**Testing built-in attributes:** in `pdk-unit`, seed them on the request with
`UnitHttpRequest::get().with_property(vec!["connection", "subject_peer_certificate"], "CN=Alice,emailAddress=alice@example.com")`.
See [[pdk-unit-tests]].

## Publishing values for downstream policies, metrics and tracing

When several policies need the same derived value (a model name, token usage, a caller identity),
compute it **once** and publish it as a property, instead of having each consumer re-parse the body:

```rust
// Producer: one namespaced, single-segment dotted key per value, as decimal/UTF-8 text.
const PROP_TOKENS_TOTAL: &[&str] = &["myorg.llm.tokens.total"];
stream.set_property(PROP_TOKENS_TOTAL, Some(total.to_string().as_bytes()));

// Consumer: parse defensively, and keep a fallback for when the producer isn't in the chain.
let total = stream.read_property(PROP_TOKENS_TOTAL)
    .and_then(|b| String::from_utf8(b).ok())
    .and_then(|s| s.trim().parse::<u64>().ok());
```

- **Contract, not coupling.** Name keys under your own prefix (`myorg.<domain>.<field>`), document
  them, and treat a missing property as "the producer didn't run" (it may be absent, ordered after
  you, or failed). Never treat it as zero.
- **Ordering.** A consumer only sees values the producer set *earlier in the chain* or, on the
  response side, earlier in the same phase. Values set during a streamed response are complete only
  after the stream ends (see [[pdk-sse-parsing]]).
- **Never publish secrets.** A property stays readable for the rest of the stream by every later
  policy and by gateway components, so publish identifiers (`client_id`, a model name), never
  tokens or credentials ([[pdk-policy-logging]]).
- **Tracing labels.** The public `tracing-flex` policy's labels support only `literal`,
  `environment` and `requestHeader` sources. To get a policy-computed value onto spans with public
  features, set a request header (`set_header("x-myorg-model", …)`) in a policy that runs **before**
  the tracing policy, and add a `requestHeader` label for it. *(MuleSoft's own internal tracing
  policy also reads properties directly, through a `value` label type with an Envoy format string
  `%FILTER_STATE(wasm.<key>:PLAIN)%`. That label type is not in the public tracing policy as of PDK
  1.10.0, so don't rely on it.)*

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-streamproperties
- Built-in-attribute pattern derived from the public `certs` PDK example (`certs/src/lib.rs`,
  `pdk-custom-policy-examples`), which reads `["connection", "subject_peer_certificate"]`.

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-streamproperties.adoc`
- **Snapshot:** 2026-08-24
- **Note:** "Publishing values for downstream policies, metrics and tracing" (2026-11-11) is
  field-derived from MuleSoft's own LLM policies. The tracing label sources are from
  https://docs.mulesoft.com/gateway/latest/policies-included-tracing (checked 2026-11-11).
