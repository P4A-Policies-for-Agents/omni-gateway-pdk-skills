---
name: pdk-metadata
description: Use when accessing metadata about the policy (PolicyMetadata), Omni Gateway instance (FlexMetadata), API instance (ApiMetadata with SLA tiers), or Anypoint Organization (PlatformMetadata) by injecting the Metadata struct into the entrypoint or wrapped functions, or the Anypoint control-plane connection (base path, service, client credentials) via the pdk::policy_context static cache.
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
    pub slas: Option<Vec<ApiSla>>,
}

pub struct ApiSla {
    pub id: String,
    pub tiers: Vec<Tier>,
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

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-metadata.adoc`
- **Snapshot:** 2026-08-24
