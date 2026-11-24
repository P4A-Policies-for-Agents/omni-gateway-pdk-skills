---
name: pdk-metadata
description: Use when accessing metadata about the policy (PolicyMetadata), Omni Gateway instance (FlexMetadata), API instance (ApiMetadata with SLA tiers, base path, the PDK 1.11 consumer endpoint URL and API Exchange asset), or Anypoint Organization (PlatformMetadata) by injecting the Metadata struct into the entrypoint or wrapped functions, or the Anypoint control-plane connection (base path, service, client credentials) via the pdk::policy_context static cache.
---

# Skill: Accessing Policy Metadata

## Topic: Implementation

This skill covers how to access metadata about the policy, Omni Gateway instance, API instance, and Anypoint Organization.

## Metadata Struct

```rust
pub struct Metadata {
    pub flex_metadata: FlexMetadata,
    pub policy_metadata: PolicyMetadata,
    pub api_metadata: ApiMetadata,
    pub platform_metadata: PlatformMetadata,
}

pub struct FlexMetadata {
    pub flex_name: String,
    pub flex_version: String,
}

pub struct PolicyMetadata {
    pub policy_name: String,
    pub policy_namespace: String,
}

pub struct ApiMetadata {
    pub id: Option<String>,
    pub name: Option<String>,
    pub version: Option<String>,
    pub base_path: Option<String>,
    pub consumer_endpoint: Option<String>, // PDK 1.11+
    pub slas: Option<Vec<ApiSla>>,
    pub asset: Option<ApiAsset>,           // PDK 1.11+ (experimental-only before)
}

pub struct ApiSla {
    pub id: String,
    pub name: String,
    pub tiers: Vec<Tier>,
}

pub struct ApiAsset {
    pub service: Service, // pdk::hl::Service for the API's Exchange asset (schema download)
}

pub struct PlatformMetadata {
    pub organization_id: String,
    pub environment_id: String,
    pub root_organization_id: String,
}
```

## Inject into Entrypoint

```rust
#[entrypoint]
async fn configure(launcher: Launcher, metadata: Metadata) -> Result<()> {
    logger::info!("Flex instance name is: {}", metadata.flex_metadata.flex_name);
    launcher.launch(on_request(filter)).await?;
    Ok(())
}
```

## Inject into Wrapped Functions

Cannot inject directly — inject into entrypoint first, then pass as reference:

```rust
#[entrypoint]
async fn configure(launcher: Launcher, metadata: Metadata) -> Result<()> {
    launcher
        .launch(on_request(|r| request_filter(r, &metadata)))
        .await?;
    Ok(())
}

async fn request_filter(_: RequestState, metadata: &Metadata) -> Flow<()> {
    let mut vec = Vec::new();
    vec.push(("flex_name".to_string(), metadata.flex_metadata.flex_name.to_string()));
    vec.push(("policy_name".to_string(), metadata.policy_metadata.policy_name.to_string()));
    Flow::Break(Response::new(201).with_headers(vec))
}
```

## Consumer Endpoint and API Asset (PDK 1.11+)

PDK 1.11 exposes two more `ApiMetadata` fields:

- **`consumer_endpoint`** is the public URL that consumers call for this API instance, as configured
  in API Manager. Use it to build absolute URLs that point back at the gateway, such as an RFC 9728
  `resource` value, a `WWW-Authenticate` `resource_metadata` link, or an OAuth redirect URI, instead
  of asking the operator to repeat the URL in the policy config.
- **`asset`** carries a `Service` for the API's Exchange asset, so a policy can download the API
  schema with an HTTP call. Before 1.11 this field existed only behind the `experimental` feature.

Both fields are `None` when the gateway doesn't supply the value (for example, an API without a
configured consumer URL, or an older Omni Gateway), so always handle the absent case. A common
pattern is to let explicit config win and fall back to metadata, ignoring blank strings:

```rust
#[entrypoint]
async fn configure(launcher: Launcher, Configuration(bytes): Configuration, metadata: Metadata) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes)?;
    let resource_url: Option<String> = config
        .resource_url                                // operator override from the schema
        .clone()
        .or_else(|| metadata.api_metadata.consumer_endpoint.clone())
        .filter(|url| !url.trim().is_empty());
    // Preserve the URL exactly, including any trailing slash: clients compare it byte for byte.

    launcher.launch(on_request(|rs| request_filter(rs, resource_url.as_deref()))).await?;
    Ok(())
}
```

In unit tests, set the field through the builder's metadata hook:
`UnitTestBuilder::new(...).metadata(|m| m.api_metadata.consumer_endpoint = Some("https://api.example.com/orders/".into()))`.

## Advanced: Anypoint control-plane context (`pdk::policy_context`)

The `Metadata` injectable above is the documented surface and covers most needs. A separate,
lower-level accessor — `pdk::policy_context::static_policy_context_cache::StaticPolicyContextCache`
— exposes the resolved **Anypoint control-plane connection** for a policy that must call the
platform itself (e.g. resolving an API-Manager endpoint). It is read via a static cache rather than
parameter injection:

```rust
use pdk::policy_context::static_policy_context_cache::StaticPolicyContextCache;

let policy_metadata = StaticPolicyContextCache::read_metadata();
let environment = policy_metadata.anypoint_environment().unwrap();
let anypoint = environment.anypoint().unwrap();

let base_path     = anypoint.base_path();       // control-plane base path
let service_name  = anypoint.service_name();     // registered control-plane service
let url           = anypoint.url();              // control-plane authority
let client_id     = anypoint.client_id();        // connected-app credentials for the platform
let client_secret = anypoint.client_secret();
```

Use this only when a policy genuinely needs to reach the Anypoint control plane; for API/flex/org
identity, prefer the injected `Metadata` struct. `client_secret()` is a credential — never log or
echo it (see [[pdk-policy-logging]]). The `anypoint_environment()`/`anypoint()` accessors return
`Option`, so handle the unresolved case instead of unwrapping blindly in production code.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-metadata
- `pdk::policy_context` accessor derived from the public `agent-policies/agent-core` PDK example
  (`pdk-custom-policy-examples`); it is not on the `Metadata` docs page.
- `consumer_endpoint`, `base_path`, `asset` and `ApiSla::name` verified against the published
  `pdk-core` 1.11.0 crate source (`policy_context/api.rs`); the docs page had not listed them as of
  the snapshot. Release notes: https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.11.0)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.11/modules/ROOT/pages/policies-pdk-configure-features-metadata.adoc`
- **Snapshot:** 2026-11-24
