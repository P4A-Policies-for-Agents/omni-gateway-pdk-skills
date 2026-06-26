---
name: pdk-examples
description: Canonical PDK (Policy Development Kit) code examples for common Omni Gateway policy features. Invoke when writing, extending, or reviewing policy code that uses any of the features listed below — e.g. authentication, JWT validation, caching, HTTP calls, body streaming, rate limiting, CORS, DataWeave evaluation, gRPC clients, timers, etc. Each example is an authoritative snippet ready to adapt into a policy's `lib.rs`, `gcl.yaml`, or `Cargo.toml`. Retrieve a specific example on demand — do not load all of them upfront.
---

# PDK feature examples

This skill ships 38 template files under `templates/`, one (or more) per PDK feature. Each file is a working example of that feature in a PDK policy, ready to copy and adapt. The templates come from the Mulesoft Flex PDK examples and are canonical — prefer them over guessing an API shape when you don't remember the exact signature.

## How to use

1. Identify the feature the user needs from the index below.
2. `Read` the matching template file from `templates/`.
3. Adapt it to the policy you're working on. Never copy the template verbatim into production — it's a shape reference, not production code.

**Don't** read templates eagerly. Only load the one(s) you actually need for the task at hand.

## Feature index

Map from feature name → file(s) under `templates/`. Features with multiple files (e.g. gRPC, HTTP call) ship a Rust example plus the config/schema/proto pieces that go with it.

### Request and response handling

- **authentication** — `authentication.rs.template`
- **body_manipulation** — `body_manipulation.rs.template`
- **body_stream** — `body_stream.rs.template`
- **control_flow** — `control_flow.rs.template`
- **header_manipulation** — `header_manipulation.rs.template`
- **request_data** — `request_data.rs.template`
- **stop_iteration** — `stop_iteration.rs.template`, `stop_iteration.toml.template`
- **outbound** — `outbound.gcl.template` (outbound-policy `gcl.yaml` shape)

### Identity, access, and policy enforcement

- **contracts** — `contracts.rs.template`
- **cors** — `cors.rs.template`
- **ip_filter** — `ip_filter.rs.template`
- **jwt** — `jwt.rs.template` (JWT validation)
- **jwt_generate** — `jwt_generate.rs.template` (JWT signing)
- **oauth2_token_introspection** — `oauth2_token_introspection.rs.template`
- **policy_violation** — `policy_violation.rs.template`

### External calls and protocols

- **http_call** — `http_call.rs.template`, `http_call.gcl.template`
- **grpc** — `grpc.rs.template`, `grpc.gcl.template`, `grpc.proto.template`, `grpc.build.template`, `grpc.toml.template`
- **websocket** — `websocket.rs.template`, `websocket.toml.template` (open beta, PDK 1.9.0; `pdk::websockets` frame decode/encode + `FilterBuilder` upgrade hooks; needs `features = ["ll", "experimental_websocket"]`)

### Validation and transformation

- **dataweave** — `dataweave.rs.template`, `dataweave.gcl.template`
- **json_validator** — `json_validator.rs.template`
- **xml_validator** — `xml_validator.rs.template`

### Rate control, concurrency, timing

- **cache** — `cache.rs.template`
- **lock** — `lock.rs.template`
- **rate_limiting** — `rate_limiting.rs.template`
- **spike_control** — `spike_control.rs.template`
- **timer** — `timer.rs.template`
- **worker_variable** — `worker_variable.rs.template`

### Observability and meta

- **logger** — `logger.rs.template`
- **metadata** — `metadata.rs.template`
- **unit_testing** — `unit_testing.ctx.template`

## Filename convention

- `<feature>.rs.template` — Rust source (policy `lib.rs` or module).
- `<feature>.gcl.template` — the `gcl.yaml` property or extension schema fragment.
- `<feature>.toml.template` — `Cargo.toml` additions (features, dependencies).
- `<feature>.proto.template`, `.build.template` — gRPC codegen inputs.
- `<feature>.ctx.template` — test-context scaffolding.

Features spanning multiple files show all the pieces needed — if you use the gRPC example, you need all five files.

## Guardrails

- The templates are **examples**, not policies themselves. They intentionally omit real business logic.
- Repo-specific conventions (handler-dispatcher pattern, workspace deps, logging rules, spec-first flow) take precedence when they conflict with a template. Use the templates for API shape only; use the repo's skills (`policy-spec`, `pdk-unit-tests`) and `docs/new-policy.md` for conventions.
- If you adapt a template, don't include the `Copyright 2026 Salesforce, Inc.` header from the source — match the copyright line used in the target policy.
