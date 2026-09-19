---
name: pdk-policy-logging
description: Use when configuring logging in PDK custom policies using the pdk::logger macros (debug, info, warn, error) that generate log messages enriched with API instance ID, policy ID, and request ID in Omni Gateway logs, and for the convention to never log or return secrets — credentials, Authorization header values, tokens, client secrets, or raw config bytes.
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

## Never log or return secrets

A policy sits on the credential path, so logs and error responses are the two easiest places to leak
a secret. The log level is user-configurable, so a `debug!` line that looks harmless in production
can be turned on — never rely on level to hide sensitive data. Rules:

- **Never log a raw credential or its container.** Do not log the `Authorization` header value
  (Basic base64 or Bearer token), passwords, API keys, client secrets, JWTs, or the raw request body
  when it may carry them. Log a non-reversible fact instead — presence, length, or a scheme name:
  ```rust
  // BAD — leaks the base64 credential into the logs:
  logger::error!("Unable to decode {}. Cause: {}", auth_value, error);
  // GOOD — enough to debug, nothing sensitive:
  logger::error!("Unable to decode Authorization credential (len={}). Cause: {}", auth_value.len(), error);
  ```
- **Never put a secret in an error you return to the client.** An `Error`/`anyhow!` value whose
  `.to_string()` becomes the HTTP response body must not embed the credential, header value, or raw
  payload. Return a generic message ("authentication failed") to the caller; keep specifics in logs,
  and only in the non-sensitive form above.
- **Never echo raw config bytes.** On a config-parse failure, do not log or return
  `String::from_utf8_lossy(&bytes)` — policy configuration can contain sensitive parameters. Log the
  parse error alone.
- **Prefer length/hash over value** whenever you must correlate a token across log lines — e.g.
  `"token extracted (len={})"`, never the token itself.

These are conventions, not compile-time checks: grep your own `debug!`/`info!`/`warn!`/`error!`
calls and returned error strings for header values, decoded credentials, and payload bytes before
shipping. See also [[pdk-coding-best-practices]] and [[pdk-metadata]] (`client_secret()`).

## Log Output Format

All log messages appear in the Omni Gateway logs in the following format:

```
[flex-gateway-envoy][<log-level>] wasm log <policy-name>.<api-instance-name> main: [policy: <policy-name>][api: <api-instance-name>][req: <request-id>] <message>
```

For more information about viewing Omni Gateway logs, see [Monitoring Omni Gateway](https://docs.mulesoft.com/gateway/latest/flex-gateway-monitoring).

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-logging

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-configure-features-logging.adoc`
- **Snapshot:** 2026-09-19
- **Note:** the "Never log or return secrets" conventions are field-derived (informed by a
  credential-redaction fix in MuleSoft's own auth policies), not from the docs page; log-level
  configurability is documented under the local-debug logging config.
