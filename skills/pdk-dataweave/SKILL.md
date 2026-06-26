---
name: pdk-dataweave
description: Use when evaluating DataWeave expressions in PDK custom policies, including schema definition with format dataweave, creating evaluators, binding variables (vars, attributes, authentication, payload), and calling eval to get Script expression results.
---

# Skill: Using DataWeave Expressions

## Topic: Implementation

This skill covers how to evaluate DataWeave expressions in PDK custom policies.

## Overview

DataWeave expression parameters are parsed and transformed into `pdk::script::Script` expressions. To evaluate:

1. Create an evaluator
2. Bind variables (vars, attributes, authentication, payload)
3. Call `eval()` to get the result

**Important:** Only call binding methods for bindings defined in the schema. If `attributes` is not used, don't call `bind_attributes`.

## Schema Definition

Define DataWeave expressions in `gcl.yaml`:

```yaml
spec:
  extends:
    - name: extension-definition
      namespace: default
  properties:
    tokenExtractor:
      type: string
      format: dataweave
      default: "#[vars.myVar]"
      bindings:
        payload:
          mimeType: text
        attributes: true
        authentication: true
        vars:
          - myVar
  required:
    - tokenExtractor
```

**Note:** Omni Gateway doesn't support binary type results. Transform binary output to string with `dw::util::Coercions::toString(binary, encoding)`.

## Evaluate DataWeave Expressions

```rust
use pdk::script::{Evaluator, HandlerAttributesBinding, Value};

async fn request_filter(
   state: RequestState,
   stream: StreamProperties,
   auth: Authentication,
   mut evaluator: Evaluator<'_>,
) {
   evaluator.bind_vars("myVar", "myVal");
   evaluator.bind_authentication(&auth.authentication());

   let state = state.into_headers_state().await;
   evaluator.bind_attributes(&HandlerAttributesBinding::new(state.handler(), &stream));

   let state = state.into_body_state().await;
   evaluator.bind_payload(&state);

   if let Ok(value) = evaluator.eval() {
       match value {
           Value::Null => info!("value was null!"),
           Value::Bool(val) => info!("value was Bool: {val}"),
           Value::Number(val) => info!("value was Number: {val}"),
           Value::String(val) => info!("value was String: {val}"),
           Value::Array(val) => info!("value was Array: {val:?}"),
           Value::Object(val) => info!("value was Object: {val:?}"),
       }
   }
}

#[entrypoint]
async fn configure(launcher: Launcher, Configuration(bytes): Configuration) -> Result<()> {
   let config: Config = serde_json::from_slice(&bytes)?;

   launcher
       .launch(on_request(|request, stream, auth| {
           request_filter(request, stream, auth, config.token_extractor.evaluator())
       }))
       .await?;
   Ok(())
}
```

## Bindings

- **vars**: Call `bind_vars(name, value)` for each variable. Unbound vars resolve to `null`.
- **attributes**: Call `bind_attributes()` with `HandlerAttributesBinding` (works with `RequestHeaderState`, `ResponseHeaderState`, HTTP call responses)
- **authentication**: Call `bind_authentication()` with `AuthenticationData`
- **payload**: Call `bind_payload()` with `RequestBodyState`, `ResponseBodyState`, or HTTP call response

## Check Expression Readiness

Some expressions can resolve before all values are bound. Check with:

```rust
evaluator.is_ready()
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-dataweave
- Example: DataWeave Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-dataweave.adoc`
- **Snapshot:** 2026-05-14
