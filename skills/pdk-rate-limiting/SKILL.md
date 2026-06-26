---
name: pdk-rate-limiting
description: Use when implementing rate limiting in PDK custom policies with RateLimitBuilder, supporting single-replica (local) and multi-replica (clustered with shared Redis storage) deployments, bucket configuration with tiers, and checking requests with is_allowed.
---

# Skill: Rate Limiting Requests

## Topic: Implementation

This skill covers how to implement rate limiting in PDK custom policies. Supports single-replica and multi-replica deployments.

## Inject RateLimitBuilder

```rust
// Buckets configuration
let buckets = vec![
    ("api", vec![Tier { requests: 100, period_in_millis: 60000 }]),     // 100 req/min
    ("user", vec![Tier { requests: 10, period_in_millis: 30000 }]),     // 10 req/30s
];

// Timer for rate limit sync (clustered mode)
let timer = clock.period(Duration::from_millis(100));

// Local mode (single-instance)
let builder = rate_limit_builder.new(builder_id);
let rate_limiter = builder.buckets(buckets).build()?;

// Clustered mode with shared storage (multi-instance)
let builder = rate_limit_builder
    .new(builder_id)
    .clustered(Rc::new(timer))
    .shared();
let rate_limiter = builder.buckets(buckets).build()?;
```

### Configuration

- `builder_id`: String identifier for the rate limiter instance (required)
- `buckets`: Rate limit tiers with requests and time windows (optional — defaults to API instance config)
- `clustered`: Enables distributed storage backends (without it, uses in-memory)
- `shared`: Enables sharing state across different policy instances
- `timer`: Periodic timer for rate limit sync (required for clustered mode)

## Check Requests Against Rate Limits

```rust
match rate_limiter.is_allowed(group, &client_key, request_amount).await {
    Ok(RateLimitResult::Allowed(_)) => Flow::Continue(()),
    Ok(RateLimitResult::TooManyRequests(_)) => Flow::Break(Response::new(429)),
    Err(e) => Flow::Break(Response::new(503)),
}
```

## Multi-Replica Shared Storage

For multi-replica deployments, configure Redis shared storage. See example `playground/config/shared-storage-redis.yaml` and `playground/docker-compose.yaml`.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-rate-limiting
- Example: Multi-Instance Rate Limiting Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-rate-limiting.adoc`
- **Snapshot:** 2026-05-14
