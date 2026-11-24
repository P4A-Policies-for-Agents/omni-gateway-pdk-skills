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
   // Verifies client_id AND client_secret against the current contract.
   pub fn authenticate(&self, client_id: &ClientId, client_secret: &ClientSecret) -> Result<ClientData, AuthenticationError>;
   // Verifies only client_id (client-id-only enforcement, no secret).
   pub fn authorize(&self, client_id: &ClientId) -> Result<ClientData, AuthorizationError>;
}
```

Both are methods on an injected `&ContractValidator` (call as `validator.authenticate(..)`), not associated functions. `authenticate` takes both credentials; `authorize` takes only the `client_id`.

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
    Ok(())
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

Poll every `INITIALIZATION_PERIOD` (100 ms) until the local copy is loaded, then release the clock
and poll every `UPDATE_PERIOD`:

```rust
async fn update_my_contracts(validator: &ContractValidator, clock: Clock) {
    let initialization_timer = clock.period(ContractValidator::INITIALIZATION_PERIOD);

    loop {
        let _ = validator.update_contracts().await;
        if validator.is_ready() {
            break;
        }
        if !initialization_timer.next_tick().await {
            return; // policy unapplied or edited
        }
    }

    let update_timer = initialization_timer
        .release()
        .period(ContractValidator::UPDATE_PERIOD);

    loop {
        if let Err(e) = validator.update_contracts().await {
            logger::warn!("Contracts update failed: {e:?}");
        }
        if !update_timer.next_tick().await {
            return;
        }
    }
}
```

End the initialization loop on `is_ready()`, not on `update_contracts().is_ok()` as the docs
example does. In the 1.11 crate, `update_contracts` also returns `Ok(())` when it skips a poll:
the period hasn't elapsed, another worker holds the lock, or the failure backoff is active. An
`is_ok()` check can therefore leave the fast initialization loop before any contracts are loaded.

**Retries are built in.** The docs say you must write your own retry handling for connectivity
errors. The crate already does: after a failed poll it sets a backoff shared by all workers,
honoring `Retry-After` on a 429, and later calls are skipped until it elapses. Polling on
`UPDATE_PERIOD` is the whole retry strategy. Don't add an application-level retry loop around
`update_contracts`.

## Authorize by Client ID Only

For client-ID enforcement without a secret, extract the ID yourself and call `authorize`:

```rust
async fn my_authorization_filter(
    state: RequestHeadersState,
    validator: &ContractValidator,
) -> Flow<()> {
    let Some(raw) = state.handler().header("client_id") else {
        return Flow::Break(Response::new(401).with_body("Invalid credentials"));
    };
    match validator.authorize(&ClientId::new(raw)) {
        Ok(_client_data) => Flow::Continue(()),
        Err(_) => Flow::Break(Response::new(403).with_body("Invalid authentication")),
    }
}
```

Use 401 for missing or malformed credentials and 403 when the contract check fails, matching the
docs convention.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-contracts
- Example: Client ID Enforcement Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.11/modules/ROOT/pages/policies-pdk-configure-features-contracts.adoc`
- **Release notes:** https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.10.0)
- **Verified against:** `pdk-contracts-lib` 1.11.0 `ContractValidator` (`is_ready`, skip paths that
  return `Ok`, shared retry backoff); where it differs from the docs page, the crate wins
- **Snapshot:** 2026-11-24
