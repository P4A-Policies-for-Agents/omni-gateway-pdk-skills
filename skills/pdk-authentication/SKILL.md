---
name: pdk-authentication
description: Use when implementing authentication data propagation between policies using the Authentication injectable to read and set authentication data with principal, client_id, client_name, and custom properties.
---

# Skill: Accessing Request Authentication Information

## Topic: Implementation

This skill covers how to read and propagate authentication data between policies using the `Authentication` injectable.

## Overview

The `Authentication` injectable provides an interface to:
- Propagate authentication data for consumption by other policies
- Consume authentication data set by another policy

## AuthenticationHandler Trait

```rust
pub trait AuthenticationHandler {
    fn authentication(&self) -> Option<AuthenticationData>;
    fn set_authentication(&self, authentication: Option<&AuthenticationData>);
}
```

## AuthenticationData Struct

```rust
pub struct AuthenticationData {
    pub principal: Option<String>,
    pub client_id: Option<String>,
    pub client_name: Option<String>,
    pub properties: Value,
}
```

## Preserve existing authentication data

Multiple authentication policies may run in the same filter chain — e.g. one sets
`client_id`/`client_name`, a later one establishes the `principal`. When your policy publishes a
result, **read back the existing `AuthenticationData`, overwrite only the field(s) you own, and
write the merged object back.** Rebuilding the struct from scratch drops whatever an upstream
policy already set.

```rust
// Correct — preserves upstream client_id/client_name/properties, sets only principal.
fn propagate_principal(authentication: &Authentication, principal: String) {
    let mut auth_data = authentication.authentication().unwrap_or_default();
    auth_data.principal = Some(principal);
    authentication.set_authentication(Some(&auth_data));
}
```

Two rules:
- **Write on the success path only.** A rejected request must not carry a principal — return
  `Flow::Break` before touching `AuthenticationData`, so downstream policies see `None`.
- **Absence of the authentication object is a chain-ordering concern, not a local-mode one** — it
  is set by an upstream policy in the filter chain, never by the control plane ([[pdk-runtime-model]]).

## Example

Read authentication data and modify client_id/client_name from custom headers:

```rust
async fn request_filter(state: RequestState, authentication: Authentication) -> Flow<()> {
    let state = state.into_headers_state().await;
    let header_handler = state.handler();

    let auth = authentication.authentication().unwrap_or_default();
    let properties = auth.properties.as_object().cloned().unwrap_or_default();

    let client_id = header_handler.header("custom_client_id_header").unwrap_or_default();
    let client_name = header_handler.header("custom_client_name_header").unwrap_or_default();

    let auth = AuthenticationData::new(
        auth.principal,
        Some(client_id),
        Some(client_name),
        properties
    );

    authentication.set_authentication(Some(&auth));
    Flow::Continue(())
}

#[entrypoint]
async fn configure(launcher: Launcher) -> Result<()> {
    let filter = on_request(|rs, auth| request_filter(rs, auth));
    launcher.launch(filter).await?;
    Ok(())
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-authentication

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-authentication.adoc`
- **Snapshot:** 2026-04-23
