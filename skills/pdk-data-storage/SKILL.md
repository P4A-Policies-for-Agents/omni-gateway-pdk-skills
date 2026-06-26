---
name: pdk-data-storage
description: Use when implementing data storage in PDK custom policies with DataStorageBuilder, including local in-memory storage for single-replica deployments and remote Redis storage for multi-replica deployments with StoreMode operations (Cas, Absent, Always).
---

# Skill: Configuring Data Storage

## Topic: Implementation

This skill covers how to implement data storage in PDK custom policies using `DataStorageBuilder`. Supports local in-memory storage for single-replica and shared Redis storage for multi-replica deployments.

## Create Storage Instances

Inject `DataStorageBuilder` in your entrypoint:

```rust
// Local storage (single-replica)
let storage = store_builder.local(namespace);

// Remote storage (multi-replica, requires shared storage config)
let storage = store_builder.remote(namespace, ttl_millis);
```

Parameters:
- `namespace`: String prefix for key isolation (required)
- `ttl_millis`: Time-to-live in milliseconds for automatic data expiration (remote only)

## Store and Retrieve Data

```rust
// Store data with CAS operation
let (current_data, version) = storage.get::<MyData>(&key).await?;
let mode = StoreMode::Cas(version);
storage.store(&key, &mode, &new_data).await?;

// Retrieve data
let (data, version) = storage.get::<MyData>(&key).await?;
```

## StoreMode Operations

- **`Cas` (Compare-And-Swap)**: Updates only if current version matches. Thread-safe conditional updates.
- **`Absent`**: Stores only if key doesn't exist. Initialize without overwriting.
- **`Always`**: Always stores, overwriting any existing value unconditionally.

## Shared Storage Configuration

For multi-replica deployments, configure shared storage (Redis). See example `playground/config/shared-storage-redis.yaml` and `playground/docker-compose.yaml` files.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-data-storage
- Example: Data Storage Stats Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-data-storage.adoc`
- **Snapshot:** 2026-04-23
