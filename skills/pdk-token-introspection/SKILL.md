---
name: pdk-token-introspection
description: Use when validating OAuth 2.0 tokens with an upstream introspection service via PDK's TokenValidatorBuilder, including configuration of introspection endpoints, authorization credentials, cache settings, scopes validation, and RFC 9728 protected resource metadata discovery.
---

# Skill: Using OAuth 2.0 Token Introspection Library Functions

## Topic: Implementation

This skill covers how to validate incoming OAuth 2.0 tokens with an upstream introspection service using PDK's Token Introspection library.

## Configure the Token Validator

Import `token_introspection` module and inject `TokenValidatorBuilder` in the entrypoint:

```rust
use pdk::token_introspection::{IntrospectionResult, ParsedToken, ScopesValidator, TokenValidator, TokenValidatorBuilder};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    validator_builder: TokenValidatorBuilder,
) -> Result<()> {
    // Create a token validator
    let mut validator_instance = validator_builder
        .new("token-cache")
        .with_path("/introspect")
        .with_authorization_value("Basic YWRtaW46YWRtaW4=") // Choose either authorization value or client ID and secret
        .with_client_id("client-id") // Client ID and secret are sent as form params in the body
        .with_client_secret("client-secret")
        .with_expires_in_attribute("exp")
        .with_max_token_ttl(600)
        .with_timeout_ms(10000)
        .with_max_cache_entries(10000)
        .with_scopes_validator(ScopesValidator::all(scopes_vector))
        .with_service(config.my_service)
        .with_protected_resource_metadata(config.auth_server_url) // Optional: enables RFC 9728 discovery
        .build();
    // ...
}
```

### TokenValidatorBuilder Methods

| Method | Parameter | Default | Description |
|--------|-----------|---------|-------------|
| `with_path` | path | `"/"` | Introspection endpoint path |
| `with_authorization_value` | value | `""` | Authorization header value for introspection requests |
| `with_client_id` | value | No default | Client ID credential for the introspection service |
| `with_client_secret` | value | No default | Client secret credential for the introspection service |
| `with_expires_in_attribute` | attr | `"exp"` | Attribute name for expiration time |
| `with_max_token_ttl` | ttl (i64) | `-1` (no cap) | Max token TTL in seconds |
| `with_timeout_ms` | timeout (u64) | `10000` | Introspection request timeout in ms |
| `with_max_cache_entries` | max_entries | `1000` | Max token cache entries |
| `with_scopes_validator` | validator | No default | Scopes validator (e.g., `ScopesValidator::all(scopes_vector)`) |
| `with_service` | service | No default (required) | OAuth2 introspection service endpoint |
| `with_protected_resource_metadata` | auth server url | No default | Enables RFC 9728 protected resource discovery. Well-known endpoint path and resource URL are derived from the API base path in PDK metadata |
| `with_protected_resource_metadata_with_properties` | auth server url, additional properties (Value) | No default | Same as above but with additional metadata properties (e.g., `scopes_supported`, `bearer_methods_supported`) |

## Validate a Token

Use `validate` to send a request to the introspection endpoint:

```rust
let auth_header = handler.header("Authorization");
let token = auth_header.split_whitespace()[1];

let result = validator
    .validate(&token)
    .await
    .map_err(PolicyError::Introspection)?;
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-token-introspection
- Example: OAuth 2.0 Token Introspection Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-token-introspection.adoc`
- **Snapshot:** 2026-05-14
