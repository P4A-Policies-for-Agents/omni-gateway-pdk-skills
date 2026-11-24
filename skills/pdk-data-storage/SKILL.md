---
name: pdk-data-storage
description: Use when implementing data storage in PDK custom policies with DataStorageBuilder, including local in-memory storage for single-replica deployments and remote Redis storage for multi-replica deployments with StoreMode operations (Cas, Absent, Always), and PDK 1.11 experimental at-rest encryption of remote stores (encrypted(true), Undecryptable errors).
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

For multi-replica deployments, configure shared storage (Redis). Configure the Redis shared storage on the gateway:
- Connected Mode: https://docs.mulesoft.com/gateway/latest/conn-shared-storage-config
- Local Mode: https://docs.mulesoft.com/gateway/latest/local-shared-storage-config

Example files: `playground/config/shared-storage-redis.yaml` (https://github.com/mulesoft/pdk-custom-policy-examples/blob/1.11.0/data-storage-stats/playground/config/shared-storage-redis.yaml) and `playground/docker-compose.yaml` in the Data Storage Stats example.

## Experimental: Encrypt a Remote Store at Rest (PDK 1.11+)

Enable `experimental_datastorage_encryption` (also part of the aggregate `experimental`), then ask
the gateway to encrypt a remote store's values:

```toml
pdk = { version = "1.11.0", features = ["experimental_datastorage_encryption"] }
```

```rust
let tokens = store_builder.remote("tokens", 3_600_000).encrypted(true);
```

- Before the handle's first write, it creates or updates the store with the encryption flag and the
  handle's TTL. If that upsert fails, `store` returns an error instead of writing plaintext.
- **It is a request, not a guarantee.** Only Redis, filesystem and distributed shared storage on
  Omni/Flex Gateway **1.15+** encrypt. Memory and Object Store backends, and older gateways, accept the
  flag and store plaintext without telling the policy. Don't treat it as your only control for
  secrets you must not persist.
- **Every handle on the store must use the same flag**, in every policy and replica. The gateway keeps
  the flag from the store's last create or update, and an unflagged handle that finds the store missing
  recreates it unencrypted.
- Turning encryption on for an existing plaintext store makes the old values unreadable. `get` returns
  `DataStorageError::Undecryptable`. Treat that value as lost, re-fetch or recompute it, and don't
  retry the read.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-data-storage
- Example: Data Storage Stats Policy
- Encryption: `pdk-data-storage-lib` 1.11.0 README and `RemoteDataStorage::encrypted` source (not yet
  on the docs page at snapshot time)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.11/modules/ROOT/pages/policies-pdk-configure-features-data-storage.adoc`
- **Snapshot:** 2026-11-24
