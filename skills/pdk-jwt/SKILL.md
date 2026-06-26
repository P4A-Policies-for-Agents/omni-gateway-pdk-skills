---
name: pdk-jwt
description: Use when validating JWT signatures with SignatureValidator, extracting tokens via TokenProvider::bearer, accessing and validating claims (audience, expiration, issuer, custom), creating tokens with JwtGenerator, or propagating claims to headers in PDK policies.
---

# Skill: Using JWT Library Functions

## Topic: Implementation

This skill covers how to use the PDK JWT library to validate JWT signatures, extract and validate claims, create tokens, and propagate claims to headers.

## Extract a JWT Token

Use `TokenProvider::bearer` to extract JWT tokens from the `Authorization: Bearer <token>` header:

```rust
use pdk::jwt::*;

async fn filter(state: RequestState) -> Flow<()> {
    let headers_state = state.into_headers_state().await;
    let token = TokenProvider::bearer(headers_state.handler())?;
    // ...
}
```

For tokens from different sources, implement custom Rust extraction logic or use a DataWeave parameter.

## Validate a JWT Signature

Initialize a `SignatureValidator` with the algorithm configuration:

```rust
pub fn new(algorithm: SigningAlgorithm, key_length: SigningKeyLength, key: String) -> Result<SignatureValidator, JWTError>
```

Supported algorithms and key lengths:

| Algorithm | Key Lengths |
|-----------|-------------|
| RSA | 256, 384, 512 |
| HMAC | 256, 384, 512 |
| ES | 256, 384 |

```rust
pub enum SigningAlgorithm { Rsa, Hmac, Es }
pub enum SigningKeyLength { Len256, Len384, Len512 }
```

Validate with the `SignatureValidation` trait:

```rust
pub trait SignatureValidation {
    fn validate(&self, token: String) -> Result<JWTClaims, JWTError>;
}
```

Example:

```rust
#[entrypoint]
async fn configure(launcher: Launcher, Configuration(configuration): Configuration) -> Result<()> {
    let config: Config = serde_json::from_slice(&configuration)?;

    let signature_validator = SignatureValidator::new(
        SigningAlgorithm::Hmac,
        SigningKeyLength::Len256,
        config.secret.clone(),
    )?;

    launcher
        .launch(on_request(|request| filter(request, &signature_validator)))
        .await?;
    Ok(())
}
```

Initialize `SignatureValidator` in `#[entrypoint]` to catch configuration errors early.

## Create a JWT Token

Use `JwtGenerator` to create tokens:

```rust
pub fn new(algorithm: SigningAlgorithm, length: SigningKeyLength, key: &str) -> Result<Self, GeneratorError>
pub fn jwt(&self, claims: JWTClaims) -> Result<String, GeneratorError>
```

Key must be provided as PEM file. Formats: RSA (pkcs#8), HMAC (plain text), ES (pkcs#8).

## Access Claims

After signature validation or using `JWTClaimsParser::parse(token)`:

> **Fixed in PDK 1.9.0 (W-22691626):** the JWT library no longer panics on an unguarded `unwrap` when parsing malformed tokens; validation now surfaces a `JWTError` instead of aborting the filter.

```rust
pub fn audience(&self) -> Option<Result<Vec<String>, JWTError>>
pub fn not_before(&self) -> Option<DateTime<Utc>>
pub fn expiration(&self) -> Option<DateTime<Utc>>
pub fn issued_at(&self) -> Option<DateTime<Utc>>
pub fn issuer(&self) -> Option<String>
pub fn jti(&self) -> Option<String>
pub fn nonce(&self) -> Option<String>
pub fn subject(&self) -> Option<String>
pub fn has_claim(&self, name: &str) -> bool
pub fn get_claim<T>(&self, name: &str) -> Option<T> where T: ValueRetrieval
pub fn has_header(&self, name: &str) -> bool
pub fn get_header(&self, name: &str) -> Option<String>
pub fn get_claims(&self) -> pdk_script::Value
pub fn get_headers(&self) -> pdk_script::Value
```

`get_claim` supports: `String`, `f64`, `Vec<String>`, `chrono::DateTime<chrono::Utc>`, `serde_json::Value`.

```rust
let some_custom_claim: Option<String> = claims.get_claim("username");
```

## Validate Claims

```rust
// Validate expiration
if let Some(exp) = claims.expiration() {
    if exp < Utc::now() {
        return Flow::Break(Response::new(400).with_body("token expired"));
    }
}

// Validate audience
if let Some(aud) = claims.audience() {
    if !aud_value.iter_mut().any(|a| a == "myAudience") {
        return Flow::Break(Response::new(400).with_body("wrong audience"));
    }
}

// Custom claim validation with DataWeave
custom_validator.bind_vars("claimSet", claims.get_claims());
let result = custom_validator.eval();
```

Custom validators can use DataWeave expressions in `gcl.yaml`:

```yaml
properties:
  customValidatorRole:
    type: string
    format: dataweave
    default: "#[vars.claimSet.role == 'superRole']"
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-jwt
- Examples: JWT Validation Policy, JWT Generation Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-jwt.adoc`
- **Snapshot:** 2026-05-14
