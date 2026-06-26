---
name: pdk-ip-filter
description: Use when filtering requests based on IP addresses using the PDK IpFilter library to create allowlists or blocklists with IPv4/IPv6 addresses and CIDR notation support.
---

# Skill: Filter IP Addresses

## Topic: Implementation

This skill covers how to filter requests based on IP addresses using the PDK IP Filter.

## Create IP Filters

Import `IpFilter` from `pdk::ip_filter` and create allowlists or blocklists:

```rust
use pdk::ip_filter::IpFilter;

// Allowlist (only specified IPs permitted)
let ip_filter = IpFilter::allow(&["192.168.1.0/24", "10.0.0.1"])?;

// Blocklist (specified IPs denied)
let ip_filter = IpFilter::block(&["192.168.1.0/24", "10.0.0.1"])?;
```

Supports:
- IPv4 and IPv6 addresses
- Individual IPs: `10.0.0.1`
- CIDR notation: `192.168.1.0/24`, `10.0.0.0/8`

## Check IP

```rust
if ip_filter.is_allowed("192.168.1.100") {
    Flow::Continue(())
} else {
    Flow::Break(Response::new(403).with_body("Forbidden"))
}
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-ip-filter
- Example: IP Filter Policy

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-ip-filter.adoc`
- **Snapshot:** 2026-05-14
