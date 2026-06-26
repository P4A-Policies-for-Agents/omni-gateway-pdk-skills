---
name: pdk-share-data-between-policies
description: Use when sharing properties between multiple policies processing the same request using the StreamProperties injectable and PropertyAccessor trait (read_property, set_property) to broadcast and consume data across policy instances.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-streamproperties

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-streamproperties.adoc`
- **Snapshot:** 2026-04-23
