---
name: pdk-policy-violations
description: Use when reporting policy violations to Anypoint Monitoring using PDK functions like generate_policy_violation and generate_policy_violation_for_client_app, or when retrieving violation information with get_policy_name, get_client_name, and get_client_id.
---

# Skill: Reporting Policy Violations

## Topic: Implementation

This skill covers how to report policy violations in a PDK custom policy implementation. Policy violations are reported to Anypoint Monitoring.

## Overview

PDK provides functions to report a policy violation. Reported policy violations appear in **Anypoint Monitoring**.

Key behaviors:
- Only **one policy violation** can be active for a request at a time
- When a new violation occurs, it **overrides** any existing violation
- If multiple policies generate violations for a single request, only one violation appears in Anypoint Monitoring
- Reporting a policy violation **does not reject** the request. To reject requests, see *Stopping Request and Response Execution*

For an example of reporting policy violations, see the **Stream Payload Policy** template.

## Generating Policy Violations

To view the current policy violation or generate a new policy violation, use these PDK functions:

```rust
/// Returns the existing policy violation of the current request. Returns None if no violation exists.
pub fn policy_violation(&self) -> Option<PolicyViolation>;

/// Generates a new policy violation for the current request. If a violation was previously generated, the previous violation is overridden.
pub fn generate_policy_violation(&self);

/// Generates a new policy violation with the client application details for the current request. This overwrites any previous policy violations.
pub fn generate_policy_violation_for_client_app<T: Into<String>, K: Into<String>>(
   &self,
   client_name: T,
   client_id: K,
);
```

## Retrieving Policy Violation Information

To retrieve information about the current policy violation, use these PDK functions:

```rust
/// The name of the policy that generated the violation
pub fn get_policy_name(&self) -> &str;

/// Client name associated with the violation, only available if a previous policy reported the client ID
pub fn get_client_name(&self) -> Option<&str>;

/// Client ID associated with the violation, only available if a previous policy reported the client ID
pub fn get_client_id(&self) -> Option<&str>;
```

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-violations
- Related: [Injecting Parameters](https://docs.mulesoft.com/pdk/latest/policies-pdk-configure-features-inject-parameters)

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `bb0f3c6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-configure-features-violations.adoc`
- **Snapshot:** 2026-04-23
