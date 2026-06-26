---
name: pdk-policy-logging
description: Use when configuring logging in PDK custom policies using the pdk::logger macros (debug, info, warn, error) that generate log messages enriched with API instance ID, policy ID, and request ID in Omni Gateway logs.
---

# Skill: Configuring Policy Logging

## Topic: Implementation

This skill covers how to configure logging in a PDK custom policy implementation. PDK provides a logging mechanism that generates log messages enriched with the API instance ID, policy ID, and request ID.

## Log Macros

Insert custom logs with the following macros by using the `pdk::logger` package:

- `logger::debug!` — Debug-level log messages
- `logger::info!` — Informational log messages
- `logger::warn!` — Warning log messages
- `logger::error!` — Error log messages

The macros behave the same as the Rust `std::format!` macro. The first parameter must be a format string literal. Use `{}` in the literal to pass parameters.

## Usage Example

```rust
use pdk::logger;

// [...]

let value = "there!";
logger::debug!("Hello there!");
logger::info!("Hello {}", value);
logger::warn!("Hello {value}");
logger::error!("Hello {}", "there!");
```

## Log Output Format

All log messages appear in the Omni Gateway logs in the following format:

```
[flex-gateway-envoy][<log-level>] wasm log <policy-name>.<api-instance-name> main: [policy: <policy-name>][api: <api-instance-name>][req: <request-id>] <message>
```

For more information about viewing Omni Gateway logs, see [Monitoring Omni Gateway](https://docs.mulesoft.com/gateway/latest/flex-gateway-monitoring).

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-logging

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `f89b114`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-logging.adoc`
- **Snapshot:** 2026-05-14
