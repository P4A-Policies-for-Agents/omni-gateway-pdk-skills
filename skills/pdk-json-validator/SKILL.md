---
name: pdk-json-validator
description: Use when validating JSON payloads with configurable limits on nesting depth, array length, object entry count, string/key size using the PDK JsonValidatorBuilder and validate_chunk for streaming bodies.
---

# Skill: Using JSON Validator Library Functions

## Topic: Implementation

This skill covers how to use PDK's JSON Validator library to enforce configurable limits on JSON payloads, such as maximum nesting depth, array length, object entry count, and string/key size. The library validates incrementally via `validate_chunk`, making it suitable for streaming bodies.

## Import

```rust
use pdk::json_validator::{JsonValidatorBuilder, ValidationError, ValidationResult};
```

## Build a JSON Validator

Create a validator with `JsonValidatorBuilder::new()`, chain configuration methods, and call `build()`:

| Method | Description |
|--------|-------------|
| `with_max_depth` | Maximum nesting depth of JSON containers (objects and arrays combined) |
| `with_max_array_length` | Maximum number of elements in a single array |
| `with_max_object_entries` | Maximum number of entries in a single object |
| `with_max_key_length` | Maximum length of an object entry name (UTF-8 byte length) |
| `with_max_string_length` | Maximum length of a string value (UTF-8 byte length) |

If you omit a `with_*` method, that limit is not applied. Call only the methods for limits you want enforced.

## Validate JSON Body Chunks

Call `validate_chunk(data, end_of_stream)` for each slice of the body. Set `end_of_stream` to `true` only on the final chunk.

Return values:
- `Ok(ValidationResult::Incomplete)` -- JSON valid so far, more chunks needed
- `Ok(ValidationResult::Complete)` -- JSON complete and valid
- `Err(ValidationError::UnableToProcessPayload)` -- payload malformed or violates a limit

When `end_of_stream` is `true` and the result is `Incomplete`, the client did not send a complete JSON value -- treat as an error.

## Full Example

```rust
use anyhow::Result;
use futures::StreamExt;
use pdk::hl::*;
use pdk::json_validator::{JsonValidatorBuilder, ValidationError, ValidationResult};

async fn request_filter(state: RequestState) -> Flow<()> {
    let headers_state = state.into_headers_state().await;

    if !headers_state.contains_body() {
        return Flow::Continue(());
    }

    let body_stream_state = headers_state.into_body_stream_state().await;
    let mut validator = JsonValidatorBuilder::new()
        .with_max_depth(10)
        .with_max_array_length(1000)
        .with_max_object_entries(1000)
        .with_max_key_length(256)
        .with_max_string_length(10000)
        .build();

    let mut stream = body_stream_state.stream();
    let mut buf = Vec::new();
    while let Some(chunk) = stream.next().await {
        buf.extend_from_slice(&chunk.into_bytes());
    }

    match validator.validate_chunk(&buf, true) {
        Ok(ValidationResult::Incomplete) => {
            Flow::Break(Response::new(400).with_body("Incomplete JSON payload"))
        }
        Ok(ValidationResult::Complete) => Flow::Continue(()),
        Err(_) => Flow::Break(Response::new(400).with_body("Invalid or disallowed JSON payload")),
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

For very large bodies, call `validate_chunk` once per chunk instead of buffering, setting `end_of_stream` to `true` only on the last chunk.

## Initialize from Policy Configuration

If limits come from a policy schema (GCL), parse the configuration and build the `JsonValidatorBuilder` in the `#[entrypoint]` function. Map your `config::Config` fields to the builder methods. Wrap the validator in `std::rc::Rc` or `std::sync::Arc` for shared ownership across closures.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-json-validator
- Example: JSON Validation Policy Example

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-json-validator.adoc`
- **Snapshot:** 2026-05-14
