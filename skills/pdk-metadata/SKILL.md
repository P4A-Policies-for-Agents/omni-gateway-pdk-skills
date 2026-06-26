---
name: pdk-metadata
description: Use when accessing metadata about the policy (PolicyMetadata), Omni Gateway instance (FlexMetadata), API instance (ApiMetadata with SLA tiers), or Anypoint Organization (PlatformMetadata) by injecting the Metadata struct into the entrypoint or wrapped functions.
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

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-metadata

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-metadata.adoc`
- **Snapshot:** 2026-05-14
