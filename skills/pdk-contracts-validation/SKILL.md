---
name: pdk-contracts-validation
description: Use when implementing client credential validation and contract enforcement using ContractValidator to authenticate and authorize client_id and client_secret pairs, extract Basic Auth credentials, poll the local contracts database, and validate SLA-tier contracts between API instances and applications.
---

# Skill: Using Contracts Validation Library Functions

## Topic: Implementation

This skill covers how to validate client credentials and implement custom client ID enforcement using the PDK Contracts Validation library.

## Overview

Contracts between API instances and applications give the application access based on SLA tiers. Client applications validate contracts by providing client ID and client secret credentials. Only one contract at a time can exist between an API instance and an application.

## Extract Client Credentials

The library is available from `pdk::contracts` module. Use `ContractValidator` to validate credentials:

```rust
impl ContractValidator {
   pub fn authenticate(client_id: &ClientId, client_secret: &ClientSecret) -> Result<ClientData, AuthenticationError>;
   pub fn authorize(client_id: &ClientId, client_secret: &ClientSecret) -> Result<ClientData, AuthorizationError>;
}
```

Both methods return `ClientData`:

```rust
pub struct ClientData {
    pub client_id: String,
    pub client_name: String,
    pub sla_id: Option<String>,
}
```

## Validate Request Authentication

Use `basic_auth_credentials()` to extract Basic Auth credentials from headers:

```rust
pub fn basic_auth_credentials(request_headers_state: &RequestHeadersState)
    -> Result<(ClientId, ClientSecret), BasicAuthError>;
```

Authentication filter example:

```rust
async fn my_authentication_filter(
  state: RequestHeadersState,
  authentication: Authentication,
  validator: &ContractValidator,
) -> Flow<()> {
    let (client_id, client_secret) = match basic_auth_credentials(&state) {
        Ok(credentials) => credentials,
        Err(e) => return Flow::Break(unauthorized_response("Invalid credentials", 401)),
    };

    let client_data = match validator.authenticate(&client_id, &client_secret) {
        Ok(client_data) => client_data,
        Err(e) => return Flow::Break(unauthorized_response("Invalid authentication", 403)),
    };

    if let Some(mut auth) = authentication.authentication() {
        auth.client_id = Some(client_data.client_id);
        auth.client_name = Some(client_data.client_name);
        authentication.set_authentication(Some(&auth));
    }

    Flow::Continue(())
}
```

## Custom Credentials Extraction

Create credentials manually with `ClientId::new()` and `ClientSecret::new()`:

```rust
let client_id = ClientId::new(raw_client_id);
let client_secret = ClientSecret::new(raw_client_secret);
```

**Note:** `ClientSecret` content is deleted after use, and `Debug` trait doesn't reveal content.

## Inject ContractValidator

```rust
#[entrypoint]
async fn configure(launcher: Launcher, validator: ContractValidator) -> Result<(), LaunchError> {
    let filter = on_request(|state, authentication| my_authentication_filter(state, authentication, &validator));
    launcher.launch(filter).await?;
    OK(())
}
```

## Poll Local Contracts Database

The `ContractValidator` maintains a local copy that must be polled periodically:

```rust
impl ContractValidator {
    pub async fn update_contracts(&self) -> Result<(), UpdateError>;
}
```

Use `ContractValidator::UPDATE_PERIOD` and `ContractValidator::INITIALIZATION_PERIOD` constants. Join polling with launcher using the `join!` macro from the `futures` crate:

```rust
#[entrypoint]
async fn configure(launcher: Launcher, clock: Clock, validator: ContractValidator) -> Result<(), LauncherError> {
    let filter = on_request(|state, authentication| my_authentication_filter(state, authentication, &validator));

    let (_, launcher_result) = join! {
        update_my_contracts(&validator, clock),
        launcher.launch(filter),
    };

    launcher_result
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-contracts
- Example: Client ID Enforcement Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-contracts.adoc`
- **Snapshot:** 2026-05-14
