---
name: pdk-share-data-between-policies
description: Use when sharing properties between multiple policies processing the same request using the StreamProperties injectable and PropertyAccessor trait (read_property, set_property) to broadcast and consume data across policy instances, or when reading built-in gateway/Envoy connection attributes (mTLS peer certificate subject, client source address, request id) that the gateway populates on the stream.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-streamproperties
- Built-in-attribute pattern derived from the public `certs` PDK example (`certs/src/lib.rs`,
  `pdk-custom-policy-examples`), which reads `["connection", "subject_peer_certificate"]`.

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-streamproperties.adoc`
- **Snapshot:** 2026-08-24
