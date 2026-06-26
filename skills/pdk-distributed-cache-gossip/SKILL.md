---
name: pdk-distributed-cache-gossip
description: Use when safely using PDK DataStorage remote backend with gossip replication in multi-replica Omni Gateway deployments, including concurrency hazards, CAS-guarded state transitions, single-initiator patterns with put_absent, tombstone propagation avoidance, and separate namespace patterns.
---

# Skill: Distributed Cache Safety with Gossip Replication

## Topic: Implementation

This skill covers how to safely use the PDK `DataStorage` remote (gossip-replicated) backend in multi-replica Omni Gateway deployments. It documents the concurrency hazards introduced by gossip replication, the patterns that mitigate them, and the rules for choosing between `StoreMode::Always`, `StoreMode::Cas`, and `StoreMode::Absent`.

## When This Skill Applies

Use this skill when:
- A policy uses `DataStorageBuilder::remote()` (distributed storage)
- Multiple gateway replicas may read/write the same cache keys concurrently
- Correctness depends on which replica "wins" a write (e.g., initiating a CIBA flow, advancing state machines, preventing duplicate side-effects)

This skill extends `pdk-data-storage` and `pdk-caching`. Read those first for basic storage APIs and local caching patterns.

## Gossip Replication Model

Remote `DataStorage` uses an eventually-consistent gossip protocol to replicate entries across replicas. Key properties:

| Property | Implication |
|----------|-------------|
| **Eventual consistency** | A write on replica A is not immediately visible on replica B |
| **Tombstone propagation** | `DELETE` creates a tombstone that gossips asynchronously — it can arrive *after* a new write to the same key, destroying the fresh entry |
| **CAS is local** | `StoreMode::Cas` checks the version on the local replica; gossip-replicated updates change the version on arrival |
| **Namespace TTL** | The `ttl_millis` parameter on `store_builder.remote()` is the sole automatic cleanup mechanism |

## Core Rule: Never DELETE Before Re-Create

**Problem:** Calling `storage.delete(key)` followed by `storage.store(key, Absent, value)` creates a gossip tombstone. If the tombstone propagates to another replica *after* the new write arrives, it kills the fresh entry.

**Rule:** When you need to replace a stale entry with a new one, use **CAS-overwrite** instead of DELETE + insert:

```rust
// WRONG — creates tombstone that can kill the new write
storage.delete(key).await;
storage.store(key, &StoreMode::Absent, &new_entry).await?;

// CORRECT — atomic overwrite, no tombstone
let (_, version) = storage.get::<MyEntry>(key).await?.unwrap();
storage.store(key, &StoreMode::Cas(version), &new_entry).await?;
```

## Pattern: Single-Initiator with `put_absent`

Use `StoreMode::Absent` when exactly one replica must perform a side-effect (e.g., sending a push notification, calling an external API) for a given key.

### Implementation

```rust
pub async fn put_absent(
    &self,
    key: &CacheKey,
    entry: &MyEntry,
) -> Result<Option<MyEntry>> {
    match self.storage.store(key.as_str(), &StoreMode::Absent, entry).await {
        Ok(()) => {
            // This replica is the confirmed first writer.
            Ok(None)
        }
        Err(DataStorageError::CasMismatch) => {
            // Key exists — classify the existing entry.
            match self.storage.get::<MyEntry>(key.as_str()).await {
                Ok(Some((existing, version))) => {
                    if existing.is_adoptable() {
                        // Active entry from another replica — adopt it.
                        Ok(Some(existing))
                    } else {
                        // Stale/terminal — CAS-overwrite (no DELETE tombstone).
                        self.storage
                            .store(key.as_str(), &StoreMode::Cas(version), entry)
                            .await?;
                        Ok(None)
                    }
                }
                _ => Err(/* key vanished — transient error */)
            }
        }
        Err(e) => Err(e.into()),
    }
}
```

### Return Value Semantics

| Return | Meaning | Caller Action |
|--------|---------|---------------|
| `Ok(None)` | This replica is the initiator | Proceed with side-effect, register with poller |
| `Ok(Some(existing))` | Another replica already initiated | Adopt `existing` state (e.g., its `auth_req_id`) |
| `Err(_)` | Storage failure | Fall back to local-only behavior or return error |

### Rules

- Only return `Ok(Some(..))` for entries that are **adoptable** (active, not expired, not terminal). Never adopt a completed/failed flow.
- On CAS-overwrite conflict (double race), re-read and try to adopt the winner.
- Never use `Ok(None)` unless the write is **confirmed** — it authorizes the caller to perform the side-effect.

## Pattern: CAS-Guarded State Transitions

Use `StoreMode::Cas` when updating shared state that multiple replicas may modify concurrently (e.g., polling results, status changes).

### Implementation with Retry

```rust
const CAS_MAX_RETRIES: u32 = 3;

async fn update_with_cas<S: DataStorage>(
    storage: &S,
    key: &str,
    update_fn: impl Fn(MyEntry) -> Option<MyEntry>,
) {
    for attempt in 0..CAS_MAX_RETRIES {
        let (entry, version) = match storage.get::<MyEntry>(key).await {
            Ok(Some(pair)) => pair,
            _ => return, // entry gone — nothing to update
        };

        let Some(updated) = update_fn(entry) else {
            return; // caller says: skip (e.g., entry already in terminal state)
        };

        match storage.store(key, &StoreMode::Cas(version), &updated).await {
            Ok(_) => return,
            Err(DataStorageError::CasMismatch) => {
                logger::debug!("CAS conflict attempt {} for key '{}' — retrying", attempt + 1, key);
                continue;
            }
            Err(e) => {
                logger::error!("CAS update error for key '{}': {:?}", key, e);
                return;
            }
        }
    }
    logger::warn!("CAS exhausted {} retries for key '{}'", CAS_MAX_RETRIES, key);
}
```

### Rules

- The `update_fn` closure must be **idempotent** — it may be called multiple times on different snapshots.
- Return `None` from `update_fn` to abort the update without error. Use this to guard against overwriting a terminal state with stale data.
- Keep `CAS_MAX_RETRIES` small (3 is typical). Exhaustion means high contention — log a warning and move on; the next tick will retry naturally.
- Always use CAS for state-machine transitions (Pending -> Approved, Pending -> Denied, etc.) to prevent a stale replica from reverting a state change that gossip already delivered.

### State Guard Example

```rust
// Poller updating last_polled_at on a Pending entry.
// If another replica already advanced to Approved via gossip,
// the closure returns None to avoid reverting the state.
update_with_cas(&store, &key, |mut entry| {
    if entry.status != Status::Pending {
        return None; // another replica advanced the state — skip
    }
    entry.last_polled_at = now;
    Some(entry)
}).await;
```

## Pattern: No Proactive Deletes on Expiry

With gossip backends, **do not delete expired entries on read**. Instead, return `None` and let the namespace TTL handle cleanup.

```rust
// WRONG for gossip backends — tombstone can kill a concurrent new write
pub async fn get(&self, key: &str) -> Option<MyEntry> {
    let (entry, _) = self.storage.get::<MyEntry>(key).await.ok()??;
    if entry.is_expired() {
        self.storage.delete(key).await; // Dangerous!
        return None;
    }
    Some(entry)
}

// CORRECT — rely on namespace TTL for cleanup
pub async fn get(&self, key: &str) -> Option<(MyEntry, String)> {
    let (entry, version) = self.storage.get::<MyEntry>(key).await.ok()??;
    if entry.is_expired() {
        // Leave in store — namespace TTL evicts it safely.
        return None;
    }
    Some((entry, version))
}
```

**Exception:** For `TokenCache` in local-only mode, proactive deletes on expired entries are safe (no gossip). The `get()` method may delete expired entries when using `store_builder.local()`.

## Pattern: Separate Storage Namespaces per Data Type

When a policy stores different types of data with different lifetimes, use **separate `DataStorage` instances** with appropriate TTLs rather than a single shared store with key-prefix namespacing.

```rust
// CORRECT — each namespace gets its own TTL
let obo_storage = store_builder.remote("obo", (DEFAULT_CACHE_TTL as u32) * 1000);
let ciba_storage = store_builder.remote("ciba", (CIBA_MAX_AGE_SECS as u32) * 1000);

// WRONG — single store with mismatched lifetimes
let shared_storage = store_builder.remote("auth", some_ttl);
// OBO tokens (5min) and CIBA entries (10min) need different TTLs
```

### Rules

- Each `remote()` call creates a distinct gossip namespace — entries in one namespace do not interfere with entries in another.
- Set `ttl_millis` to match the maximum expected lifetime of entries in that namespace.
- Use descriptive namespace names (`"obo"`, `"ciba"`, `"intask"`) for operational clarity.

## Pattern: Schema Configuration for Distributed Mode

Expose a boolean flag in the policy schema so operators can opt in to distributed storage:

```yaml
# definition/gcl.yaml
spec:
  properties:
    distributed:
      type: boolean
      title: Distributed
      description: Share cache between replicas.
      default: false
```

In the entrypoint, branch on this flag to create the appropriate storage backend:

```rust
#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    store_builder: DataStorageBuilder,
    // ...
) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes)?;
    let use_remote = config.distributed.unwrap_or(false);

    if use_remote {
        let storage = store_builder.remote("my-ns", ttl_millis);
        launch_policy(launcher, &storage, &config).await?;
    } else {
        let storage = store_builder.local("my-ns");
        launch_policy(launcher, &storage, &config).await?;
    }
    Ok(())
}
```

### Rules

- Default to `false` (local mode). Distributed mode requires shared storage infrastructure on the gateway.
- The `launch_policy` helper must be generic over `S: DataStorage` to accept either backend.
- Keep the branch only in `configure()` — all downstream code should be backend-agnostic via the `DataStorage` trait.

## Error Handling: `CacheError::CasConflict`

Add a dedicated error variant for CAS failures so callers can distinguish retriable conflicts from hard errors:

```rust
pub enum CacheError {
    // ... existing variants ...

    /// A Compare-And-Swap write was rejected because another writer
    /// concurrently modified the entry. Callers should re-read and retry.
    CasConflict,
}
```

Map `DataStorageError::CasMismatch` to this variant:

```rust
self.storage
    .store(key, &StoreMode::Cas(version), &entry)
    .await
    .map_err(|e| match e {
        DataStorageError::CasMismatch => CacheError::CasConflict,
        other => CacheError::StorageError(format!("CAS update failed: {:?}", other)),
    })?;
```

## Testing Distributed Cache Behavior

### MockDataStorage for Unit Tests

```rust
struct MockDataStorage {
    data: Mutex<HashMap<String, Vec<u8>>>,
}

impl DataStorage for MockDataStorage {
    async fn store<T: serde::Serialize>(
        &self,
        key: &str,
        mode: &StoreMode,
        item: &T,
    ) -> Result<(), DataStorageError> {
        let bytes = bincode::serialize(item)
            .map_err(DataStorageError::Serialization)?;
        let mut map = self.data.lock().unwrap();
        match mode {
            StoreMode::Always => { map.insert(key.to_string(), bytes); }
            StoreMode::Absent => {
                if map.contains_key(key) {
                    return Err(DataStorageError::CasMismatch);
                }
                map.insert(key.to_string(), bytes);
            }
            StoreMode::Cas(_version) => {
                // Simplified — always succeeds in basic mock.
                // For race-condition tests, add version tracking.
                map.insert(key.to_string(), bytes);
            }
        }
        Ok(())
    }

    async fn get<T: serde::de::DeserializeOwned>(
        &self,
        key: &str,
    ) -> Result<Option<(T, String)>, DataStorageError> {
        let map = self.data.lock().unwrap();
        match map.get(key) {
            Some(bytes) => {
                let item: T = bincode::deserialize(bytes)
                    .map_err(DataStorageError::Serialization)?;
                Ok(Some((item, "1".to_string())))
            }
            None => Ok(None),
        }
    }

    async fn delete(&self, key: &str) -> Result<(), DataStorageError> {
        self.data.lock().unwrap().remove(key);
        Ok(())
    }

    async fn delete_all(&self) -> Result<(), DataStorageError> {
        self.data.lock().unwrap().clear();
        Ok(())
    }

    async fn get_keys(&self) -> Result<Vec<String>, DataStorageError> {
        Ok(self.data.lock().unwrap().keys().cloned().collect())
    }
}
```

### CAS Race Simulation

To test CAS conflict handling, create a mock that injects `CasMismatch` on demand:

```rust
struct CasMockStorage {
    inner: MockDataStorage,
    fail_next_cas: AtomicBool,
}

impl DataStorage for CasMockStorage {
    async fn store<T: Serialize>(
        &self, key: &str, mode: &StoreMode, item: &T,
    ) -> Result<(), DataStorageError> {
        if let StoreMode::Cas(_) = mode {
            if self.fail_next_cas.swap(false, Ordering::SeqCst) {
                return Err(DataStorageError::CasMismatch);
            }
        }
        self.inner.store(key, mode, item).await
    }
    // delegate other methods to inner...
}
```

## Quick Reference: Rules Summary

| Rule | Description |
|------|-------------|
| Never DELETE before re-create | Use CAS-overwrite to avoid gossip tombstone races |
| No proactive deletes on expiry | Let namespace TTL handle cleanup on remote backends |
| Use `Absent` for single-initiator | Guarantees exactly one replica performs a side-effect |
| Use `Cas` for state transitions | Prevents stale replicas from reverting state changes |
| State guards in update closures | Return `None` to skip updates when state has advanced |
| Separate namespaces per data type | Each type gets its own TTL and gossip isolation |
| Keep CAS retries small (3) | Exhaustion = high contention; next tick retries naturally |
| Default distributed to `false` | Requires shared storage infrastructure on the gateway |
| Dedicated `CasConflict` error | Enables callers to distinguish retriable vs hard errors |
| Generic over `DataStorage` trait | Policy logic must be backend-agnostic; branch only in `configure()` |

## Documentation Reference

- Prerequisite: `pdk-data-storage` skill (basic `DataStorage` API, `StoreMode` operations)
- Prerequisite: `pdk-caching` skill (local cache patterns, key design, TTL management)
- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-data-storage

## Source Ref

- **Snapshot:** 2026-04-08
- **Derived from:** migration of token caches from `pdk::cache::Cache` to `pdk::data_storage::DataStorage` with gossip-safe CAS patterns
