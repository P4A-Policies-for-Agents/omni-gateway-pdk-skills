---
name: pdk-caching
description: Use when implementing shared state between Envoy workers using the Cache mechanism, including cache configuration with CacheBuilder, TTL management, expiration handling, key construction rules, rate limiting with sliding windows, and response caching patterns.
---

# Skill: Sharing Data Between Workers and Configuring Caching

## Topic: Implementation

This skill covers how to share global state between Envoy workers and configure caching in PDK custom policies.

## Overview

Envoy spawns many workers to handle requests. A single worker handles all stages of a specific request. Each worker processes one request at a time — use `RefCell` for per-worker global state without concurrency risk.

Use the `Cache` mechanism for shared global state across all workers.

## Configure a Cache

Inject `CacheBuilder` in the `#[entrypoint]` function, provide an ID and max elements, then pass the built cache to filters:

```rust
use pdk::cache::{Cache, CacheBuilder, CacheError};

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
    cache_builder: CacheBuilder,
) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes)?;

    let cache = cache_builder
        .new("my-cache".to_string())
        .max_entries(100)
        .build();

    let filter = on_request(|request_state| request_filter(request_state, &config, &cache))
        .on_response(|response_state, request_data| {
            response_filter(response_state, request_data, &config, &cache)
        });

    launcher.launch(filter).await?;
    Ok(())
}
```

**Rules:**
- Always call `.max_entries()` — never leave it unset.
- Use a descriptive cache name string to isolate namespaces when multiple caches exist.
- Build the cache once in `configure()` and pass it by reference to filters.

## Cache Interface

```rust
pub trait Cache {
    fn save(&self, key: &str, value: Vec<u8>) -> Result<(), CacheError>;
    fn get(&self, key: &str) -> Option<Vec<u8>>;
    fn delete(&self, key: &str) -> Option<Vec<u8>>;
    fn purge(&self);
}
```

**Important:** The PDK `Cache` is shared across all Envoy workers. Different workers write concurrently — no guarantee the value is unchanged between a `get` and a subsequent `save`. Use "last writer wins" semantics for idempotent data (e.g., tokens). For atomic read-modify-write, use `DataStorage` with CAS instead (see `pdk-data-storage` skill).

## Cache Key Rules

### Simple Keys

For single-purpose caches, use a descriptive static key or the request path:

```rust
const WINDOW_CACHE_KEY: &str = "token-rate-limit-window";

// Or use the request path as key for response caching:
let path = headers_state.path();
cache.get(path.as_str());
```

### Structured Namespaced Keys

When a single cache stores multiple types of data, use a `{prefix}::{field1}::{field2}` format with a type prefix to avoid collisions:

```rust
// Key format: "intask::{user_id}::{audience}::{scopes}"
let key = format!("intask::{}::{}::{}", user_id, audience, scopes_joined);

// Key format: "obo::{token_hash}::{audience}::{scopes}"
let key = format!("obo::{}::{}::{}", token_hash, audience, scopes_joined);
```

### Key Consistency Rules

- **Sort variable-order inputs** before including them in the key. For example, sort scopes so `"read write"` and `"write read"` produce the same key:

```rust
let mut scopes: Vec<String> = scopes_str.split_whitespace().map(String::from).collect();
scopes.sort();
let scopes_joined = scopes.join(" ");
```

- **Hash sensitive values** (tokens, PII) instead of using them directly in keys. Use a fixed-length hash to keep keys compact and avoid leaking secrets:

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

fn hash_token(token: &str) -> String {
    let mut hasher = DefaultHasher::new();
    token.hash(&mut hasher);
    format!("{:016x}", hasher.finish())
}

// Use hash in key instead of raw token
let key = format!("obo::{}", hash_token(subject_token));
```

- **Keys must be deterministic** — same inputs must always produce the same key.

## Expiration Management

The PDK `Cache` has **no built-in TTL**. You must manage expiration in application logic.

### Embed Expiry in Cached Value

Store the expiry timestamp as a prefix in the cached bytes:

```rust
const EXPIRY_PREFIX_LEN: usize = 8;

// Store: prepend 8-byte big-endian expiry timestamp
fn cache_store(cache: &dyn Cache, key: &str, value: &[u8], expires_at: u64) -> Result<(), CacheError> {
    let mut data = Vec::with_capacity(EXPIRY_PREFIX_LEN + value.len());
    data.extend_from_slice(&expires_at.to_be_bytes());
    data.extend_from_slice(value);
    cache.save(key, data)
}

// Read: check expiry, delete if expired
fn cache_get(cache: &dyn Cache, key: &str) -> Option<Vec<u8>> {
    let data = cache.get(key)?;

    if data.len() <= EXPIRY_PREFIX_LEN {
        // Corrupt entry — clean up
        cache.delete(key);
        return None;
    }

    let expires_at = u64::from_be_bytes(data[..EXPIRY_PREFIX_LEN].try_into().unwrap());
    let now = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs();

    if expires_at <= now {
        // Expired — clean up
        cache.delete(key);
        return None;
    }

    Some(data[EXPIRY_PREFIX_LEN..].to_vec())
}
```

### Alternative: Expiry Inside Serialized Struct

For JSON-serialized values, include the expiry as a field:

```rust
#[derive(Serialize, Deserialize)]
struct CachedResponse {
    valid_until: DateTime<Local>,
    status_code: u32,
    headers: Vec<(String, String)>,
    body: Vec<u8>,
}

impl CachedResponse {
    fn has_expired(&self, now: &DateTime<Local>) -> bool {
        self.valid_until.lt(now)
    }
}
```

### TTL Calculation Rules

When caching tokens or time-bounded data:

```rust
const DEFAULT_MAX_TTL: u64 = 300;     // 5 minutes
const MIN_CACHE_TTL: u64 = 10;        // Don't cache if < 10s remaining
const SAFETY_MARGIN: f64 = 0.9;       // Expire cache before token

fn calculate_cache_ttl(token_expires_at: u64, max_ttl: u64) -> Option<u64> {
    let now = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs();

    if token_expires_at <= now {
        return None; // Already expired
    }

    let remaining = token_expires_at - now;
    let safe_remaining = (remaining as f64 * SAFETY_MARGIN) as u64;

    if safe_remaining < MIN_CACHE_TTL {
        return None; // Too short-lived to cache
    }

    Some(safe_remaining.min(max_ttl))
}
```

**Rules:**
- Always cap TTL at a configured maximum (default 300 seconds).
- Apply a safety margin (e.g., 90%) so the cache entry expires before the token.
- Don't cache data with less than ~10 seconds remaining.
- Never cache already-expired data.

## Invalidation Rules

### Lazy Expiration (Preferred)

Check expiry on every `get()`. Delete expired or corrupt entries immediately:

```rust
fn get_cached(cache: &dyn Cache, key: &str) -> Option<MyData> {
    let data = cache.get(key)?;

    // Deserialize
    let entry: MyData = match serde_json::from_slice(&data) {
        Ok(v) => v,
        Err(e) => {
            logger::warn!("Corrupt cache entry for '{}': {} — removing", key, e);
            cache.delete(key);
            return None;
        }
    };

    // Check logical expiration
    if entry.has_expired() {
        logger::debug!("Cache entry expired for '{}'", key);
        cache.delete(key);
        return None;
    }

    Some(entry)
}
```

**Rules:**
- Always validate cached data on read — check both deserialization and expiry.
- On corruption (invalid bytes, failed deserialization), delete the entry and treat as a miss.
- On expiry, delete the entry to free space.
- Never let a cache error block the request — log a warning and fall through to the backend.

## Caching Responses (Request/Response Pattern)

Use `Flow` and request data to pass state from the request filter to the response filter:

```rust
enum CachingData {
    SaveResponse(String),  // cache key (e.g., request path)
    IgnoreCache,
}

async fn request_filter(
    request_state: RequestState,
    config: &Config,
    cache: &impl Cache,
) -> Flow<CachingData> {
    let headers_state = request_state.into_headers_state().await;
    let path = headers_state.path();

    // Try cache first
    match cache.get(path.as_str()) {
        Some(data) => {
            match serde_json::from_slice::<CachedResponse>(&data) {
                Ok(cached) if !cached.has_expired(&Local::now()) => {
                    // Cache HIT — return cached response, skip backend
                    Flow::Break(cached.into())
                }
                Ok(_) => {
                    // Expired — proceed to backend, save new response
                    cache.delete(path.as_str());
                    Flow::Continue(CachingData::SaveResponse(path))
                }
                Err(_) => {
                    // Corrupt — clean up, proceed to backend
                    cache.delete(path.as_str());
                    Flow::Continue(CachingData::SaveResponse(path))
                }
            }
        }
        None => {
            // Cache MISS — proceed to backend
            Flow::Continue(CachingData::SaveResponse(path))
        }
    }
}

async fn response_filter(
    response_state: ResponseState,
    caching_data: RequestData<CachingData>,
    config: &Config,
    cache: &impl Cache,
) {
    if let RequestData::Continue(CachingData::SaveResponse(path)) = caching_data {
        let headers_state = response_state.into_headers_state().await;
        let status_code = headers_state.status_code();
        let headers = headers_state.handler().headers();

        let body_state = headers_state.into_body_state().await;
        let body = body_state.handler().body();

        let response = CachedResponse {
            valid_until: Local::now() + Duration::hours(1),
            status_code,
            headers,
            body,
        };

        if let Ok(serialized) = serde_json::to_vec(&response) {
            if let Err(e) = cache.save(path.as_str(), serialized) {
                logger::warn!("Failed to cache response: {e}");
            }
        }
    }
}
```

## Caching for Rate Limiting (Sliding Window)

Use the cache to store a time-window counter:

```rust
#[derive(Deserialize, Serialize)]
struct Window {
    expiration: SystemTime,
    token_count: usize,
}

fn validate(&self, tokens: usize) -> Result<(), RateLimitError> {
    let now = SystemTime::now();

    // Get or create window
    let mut window = self.cache.get(WINDOW_CACHE_KEY)
        .and_then(|bytes| serde_json::from_slice::<Window>(&bytes).ok())
        .filter(|w| now < w.expiration)      // Discard expired window
        .unwrap_or(Window {
            expiration: now + self.window_period,
            token_count: 0,
        });

    window.token_count += tokens;

    // Save updated window
    let serialized = serde_json::to_vec(&window)?;
    self.cache.save(WINDOW_CACHE_KEY, serialized)?;

    if window.token_count > self.maximum_tokens {
        return Err(RateLimitError::Exceeded);
    }

    Ok(())
}
```

**Note:** This pattern is not atomic — concurrent workers may race. For strict rate limiting, use `DataStorage` with CAS operations instead.

## Testing Cache Behavior

### Unit Test Mock

Create a simple `MockCache` for unit tests:

```rust
#[cfg(test)]
mod tests {
    use std::collections::HashMap;
    use std::sync::Mutex;
    use pdk::cache::{Cache, CacheError};

    struct MockCache {
        data: Mutex<HashMap<String, Vec<u8>>>,
    }

    impl MockCache {
        fn new() -> Self {
            Self { data: Mutex::new(HashMap::new()) }
        }
    }

    impl Cache for MockCache {
        fn get(&self, key: &str) -> Option<Vec<u8>> {
            self.data.lock().unwrap().get(key).cloned()
        }
        fn save(&self, key: &str, value: Vec<u8>) -> Result<(), CacheError> {
            self.data.lock().unwrap().insert(key.to_string(), value);
            Ok(())
        }
        fn delete(&self, key: &str) -> Option<Vec<u8>> {
            self.data.lock().unwrap().remove(key)
        }
        fn purge(&self) {
            self.data.lock().unwrap().clear();
        }
    }

    #[test]
    fn test_cache_store_and_retrieve() {
        let cache = MockCache::new();
        cache.save("key1", b"value1".to_vec()).unwrap();
        assert_eq!(cache.get("key1"), Some(b"value1".to_vec()));
    }

    #[test]
    fn test_expired_entry_cleanup() {
        let cache = MockCache::new();
        // Store an entry with an already-past expiry
        let expired_at: u64 = 1000; // well in the past
        let mut data = Vec::new();
        data.extend_from_slice(&expired_at.to_be_bytes());
        data.extend_from_slice(b"stale-data");
        cache.save("key1", data).unwrap();

        // cache_get should return None and delete the entry
        assert!(cache_get(&cache, "key1").is_none());
        assert!(cache.get("key1").is_none());
    }

    #[test]
    fn test_cross_worker_roundtrip() {
        let cache = MockCache::new();
        // Worker A writes
        cache.save("shared-key", b"shared-value".to_vec()).unwrap();
        // Worker B reads
        assert_eq!(cache.get("shared-key"), Some(b"shared-value".to_vec()));
    }
}
```

### Integration Test Pattern

Use `pdk_test` with `max_entries` to verify eviction behavior:

```rust
#[pdk_test]
async fn caching() -> anyhow::Result<()> {
    let policy_config = PolicyConfig::builder()
        .name(POLICY_NAME)
        .configuration(serde_json::json!({
            "max_cached_values": 1,  // Low limit to test eviction
            "start_hour": 0,
            "end_hour": 23
        }))
        .build();

    // ... setup Flex + backend ...

    // First request: cache MISS, hits backend
    let response = reqwest::get(format!("{flex_url}/route_1")).await?;
    assert_eq!(response.text().await?, "Value 1");

    // Second request: cache HIT, does NOT hit backend
    let response = reqwest::get(format!("{flex_url}/route_1")).await?;
    mock.assert_hits(1); // Still 1 — served from cache

    // Different route: evicts first entry (max_entries=1)
    let response = reqwest::get(format!("{flex_url}/route_2")).await?;

    // Original route: cache MISS again (evicted)
    let response = reqwest::get(format!("{flex_url}/route_1")).await?;
    mock.assert_hits(2); // Backend hit again

    Ok(())
}
```

## Policy Schema for Cache Configuration

Expose cache parameters in the policy definition (`definition/gcl.yaml`):

```yaml
spec:
  properties:
    max_cached_values:
      type: integer
      default: 100
    cache_ttl_seconds:
      type: integer
      default: 300
  required:
    - max_cached_values
```

## Quick Reference: Rules Summary

| Rule | Description |
|------|-------------|
| Always set `max_entries` | Never leave the cache builder without a size limit |
| Embed expiry in values | PDK Cache has no native TTL — manage expiration yourself |
| Validate on read | Check deserialization and expiry; delete corrupt/expired entries |
| Cap TTL with safety margin | `min(remaining * 0.9, max_ttl)`, skip if < 10s left |
| Deterministic keys | Same inputs must always produce the same key |
| Hash sensitive data in keys | Never store raw tokens or PII in cache keys |
| Namespace keys with prefixes | Use `{type}::{fields}` to avoid collisions in shared caches |
| Never block on cache errors | Log warnings, fall through to backend |
| Use `Cache` for idempotent data | Tokens, responses — "last writer wins" is acceptable |
| Use `DataStorage` with CAS for counters | When atomic read-modify-write is required |

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-caching
- Example: Data Caching Policy (`pdk-custom-policy-examples/data-caching`)
- Example: Token Rate Limiting (`pdk-custom-policy-examples/ai-basic-token-rate-limiting`)

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-caching.adoc`
- **Snapshot:** 2026-04-23
- **Enhanced:** 2026-03-28 — added patterns from `pdk-custom-policy-examples`
