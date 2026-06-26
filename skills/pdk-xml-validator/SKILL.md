---
name: pdk-xml-validator
description: Use when validating XML payloads with PDK's XmlValidatorBuilder to enforce configurable limits on element depth, attribute counts, child counts, text/attribute/comment lengths via streaming body validation, including integration with policy configuration and request filter rejection patterns.
---

# Skill: Using XML Validator Library Functions

## Topic: Implementation

This skill covers how to use PDK's XML Validator library to enforce configurable limits on XML payloads, such as maximum element depth, attribute counts, child counts, and text/attribute/comment lengths. The library validates by streaming the incoming XML body.

## Import

```rust
use pdk::xml_validator::XmlValidatorBuilder;
```

## Build an XML Validator

Create a validator with `XmlValidatorBuilder::new()`, chain configuration methods, and call `build()`:

| Method | Description |
|--------|-------------|
| `with_max_depth` | Maximum nesting depth of elements in the document |
| `with_max_attribute_count` | Maximum number of attributes on a single element |
| `with_max_child_count` | Maximum number of child nodes of a single element |
| `with_max_text_length` | Maximum length of text node content (UTF-8 byte length) |
| `with_max_attribute_length` | Maximum length of a single attribute value (UTF-8 byte length) |
| `with_max_comment_length` | Maximum length of comment content (UTF-8 byte length) |

If you omit a `with_*` method, that limit is not applied.

## Validate an XML Request Body Stream

Use `validate_stream` with the streaming body state. The method consumes the stream, succeeds if the document is well-formed and within limits, and returns an error if the XML is invalid or breaks a constraint.

## Full Example

```rust
use anyhow::Result;
use pdk::hl::*;
use pdk::xml_validator::XmlValidatorBuilder;

async fn request_filter(state: RequestState) -> Flow<()> {
    let headers_state = state.into_headers_state().await;

    if !headers_state.contains_body() {
        return Flow::Continue(());
    }

    let body_stream_state = headers_state.into_body_stream_state().await;
    let validator = XmlValidatorBuilder::new()
        .with_max_depth(10)
        .with_max_attribute_count(10)
        .with_max_child_count(50)
        .with_max_text_length(1024)
        .with_max_attribute_length(256)
        .with_max_comment_length(1024)
        .build();

    match validator.validate_stream(body_stream_state.stream()).await {
        Ok(()) => Flow::Continue(()),
        Err(_) => Flow::Break(Response::new(400).with_body("Invalid or disallowed XML payload")),
    }
}

#[entrypoint]
async fn configure(launcher: Launcher, Configuration(_configuration): Configuration) -> Result<()> {
    launcher
        .launch(on_request(|request| request_filter(request)))
        .await?;
    Ok(())
}
```

## Initialize from Policy Configuration

If limits come from a policy schema (GCL), parse the configuration and build the `XmlValidatorBuilder` in the `#[entrypoint]` function. Map your `config::Config` fields to the builder methods. Wrap the validator in `std::rc::Rc` or `std::sync::Arc` for shared ownership across closures.

## Interpret Validation Errors

If `validate_stream` returns `Err`, reject the payload. The XML may be malformed or violate a configured limit. Log the error at debug or warn level for operational detail. Return a generic client-facing message.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-xml-validator
- Example: XML Validation Policy Example

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-xml-validator.adoc`
- **Snapshot:** 2026-05-14
