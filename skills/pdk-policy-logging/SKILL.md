---
name: pdk-policy-logging
description: Use when configuring logging in PDK custom policies using the pdk::logger macros (debug, info, warn, error) that generate log messages enriched with API instance ID, policy ID, and request ID in Omni Gateway logs, and for the convention to never log or return secrets — credentials, Authorization header values, tokens, client secrets, or raw config bytes — plus the undocumented guaranteed-delivery idiom of logging at error! with a fixed prefix to bypass log-level filtering.
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

## Guaranteed-delivery logging (undocumented / internal idiom)

> **Caveat.** This is an internal MuleSoft convention, **not** a documented PDK feature. PDK has no
> API to emit a message regardless of the configured log level; this idiom relies on operators
> leaving `error!` (and `warn!`) unfiltered. Treat it as a workaround, not a supported channel.

When a policy must emit an event that downstream log tooling (e.g. a Fluent Bit pipeline) will scrape
even when the policy's log level is turned up, MuleSoft's own policies deliberately log at `error!`
with a fixed, greppable prefix, because in practice `error!`/`warn!` are the levels operators leave
enabled:

```rust
const ALERT_LOG_PREFIX: &str = "[alertLog]";

// Emitted at error! so it survives level filtering; the prefix makes it machine-scrapable.
// log-lint: allow-level
logger::error!("{ALERT_LOG_PREFIX} {json}");
```

Constraints if you use this:

- **Never put a secret in the payload** — this line is high-visibility by design; the "Never log or
  return secrets" rules above still apply in full.
- **Use a fixed, unique prefix** so consumers can match it precisely, and keep the payload structured
  (e.g. JSON) rather than free text.
- **Reserve it for genuine always-surface events** (alerts), not routine tracing — abusing `error!`
  for ordinary logs makes real errors harder to find and pollutes error dashboards.

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
  configurability is documented under the local-debug logging config. The "Guaranteed-delivery
  logging" `error!`-prefix idiom is undocumented — observed in MuleSoft's internal
  `microgateway-hybrid-policies` (`libs/alerts-core`).
