---
name: pdk-runtime-model
description: Use before writing or reviewing any PDK custom policy — the runtime constraints that decide what "correct" and "fast" mean. Covers the proxy-wasm sandbox (no fs/net/threads, async-only, no panic on the hot path), latency as a product constraint, stream-vs-buffer, multi-replica no-shared-memory, control-plane-independent startup, local-mode resilience (never panic on missing ApiContext), fail-open vs fail-closed as a spec decision, and the orchestrator-vs-testable-helper handler pattern that keeps policy logic unit-testable.
---

# Skill: PDK policy runtime model

## Topic: Architecture & constraints

Every PDK custom policy runs inside **Envoy** as a **proxy-wasm** filter. The facts below constrain
what "correct" and "fast" mean for a policy. Read this before designing one — several of these are
spec-level decisions, not implementation details, and getting them wrong is expensive to unwind
later.

This skill is the *why*. For the *how* (code patterns, error handling, telemetry), see
[[pdk-coding-best-practices]]; for the APIs it references, see the per-feature skills linked at the
end.

## When this skill applies

Use it when:
- Scoping or designing a new policy (what it may and may not do at runtime).
- Reviewing a policy for sandbox violations, panics, blocking calls, or unbounded buffering.
- Deciding fail-open vs fail-closed, or how the policy behaves with no control-plane connection.
- Structuring filter functions so the logic is unit-testable without PDK machinery.

## Runtime facts

### proxy-wasm sandbox
Policies compile to `wasm32-wasip1` and run in Envoy's proxy-wasm sandbox. **No filesystem, no
arbitrary outbound network, no threads.** All I/O must be async via PDK's runtime (`pdk::hl::*`):

- No `println!` / `eprintln!` — use `pdk::logger` ([[pdk-policy-logging]]).
- No `std::fs`, no `std::net` — outbound calls go through the PDK `HttpClient` ([[pdk-http-call]]).
- No blocking calls; everything on the request path is `async`.
- **No `panic!` on the hot path** — a panic crashes the filter instance. Never `unwrap()` /
  `expect()` on anything that can fail at runtime.

### Latency is a product constraint
Every policy sits on the request path, and many of them chain. Synchronous work above a few
milliseconds on the hot path needs explicit justification in the spec. Budget for the **whole
chain**, not just your policy.

### Stream when possible; buffer when you must
Prefer `RequestBodyStreamState` / `ResponseBodyStreamState` over buffering the full body
([[pdk-request-headers-bodies]]). Buffering is acceptable only when the policy genuinely needs the
whole payload (JSON-schema validation, full-document transcoding) — and even then an upper bound
belongs in the spec. Large bodies under buffered filters stall the request and balloon memory. For
SSE and other streaming MIME types, **streaming is mandatory** — buffering breaks the streaming
contract from the client's point of view.

### Multiple replicas, no shared memory
Each Envoy replica loads its own filter instances. There is **no shared in-process state across
replicas.** A `HashMap` inside a policy is per-filter-instance and invisible everywhere else.
Anything needing cross-replica consistency (rate limits, session counts, distributed locks) must go
through a PDK-backed store — and that introduces its own concurrency hazards
([[pdk-distributed-cache-gossip]], [[pdk-data-storage]], [[pdk-caching]]).

### Control-plane-independent startup
A policy MUST load and respond to requests from t=0 even if the control plane is unreachable. It may
*degrade* when a control-plane artifact is missing (e.g. a schema not yet fetched → return a defined
error), but it cannot *fail to start*. The degraded-state behavior is spec content
([[pdk-policy-spec]]), not an accident of the code.

### Local-mode resilience — never panic on missing context
Policies MUST work in **local mode**, where the gateway runs without a control-plane connection. In
local mode only `PolicyMetadata` (policy name, namespace, filter name) and `FlexMetadata` (gateway
name, version) are present. Everything else in the `ApiContext` is **absent**:

- no `environment` (no Anypoint credentials, no org/env IDs),
- no `api` info (no Exchange asset, no base path),
- no `tiers`,
- no `identityManagement`,
- no `platformPolicyIDs`.

When any of this is missing, the default behavior is: **log a warning (`pdk::logger::warn`) and let
the request continue (`Flow::Continue`).** The policy must not `panic!`, must not `unwrap()` an
`Option`/`Result` derived from context, and must not return an opaque 500. If the spec defines a
stricter failure mode for a specific missing context, document that override in the spec's Behavior
section — but the implementation still must not panic.

Note: the **authentication object is NOT a local-mode concern.** It's set by an upstream
authentication policy in the filter chain, not by the control plane. Its absence is a
chain-ordering question, not a deployment-mode question. See [[pdk-metadata]] and
[[pdk-authentication]].

Test the local-mode path with pdk-unit's `.local_mode()` builder (feature
`experimental_local_mode`) — it sets `anypoint: None` and strips the `ApiContext`, matching a real
disconnected gateway. Without it, pdk-unit populates fake-but-present Anypoint data, so tests pass
even when the policy would panic in a real local deployment. See [[pdk-unit-tests]].

### Fail-open vs fail-closed is a spec-level decision
When an external dependency is unreachable (IdP, contract service, upstream), the spec must **name**
the failure mode and defend it. A PII-detection policy failing open is almost certainly wrong; a
telemetry policy failing closed is almost certainly wrong. Don't default silently — make it explicit
in [[pdk-policy-spec]].

## The handler pattern — keep policy logic testable

Filter functions mix two concerns: PDK **state transitions** (`into_headers_state()`,
`into_body_state()`) and **business logic** (parse, validate, transform). State transitions need the
async PDK runtime; business logic doesn't. Keeping them separate is what makes a policy unit-testable
without standing up PDK machinery.

Handlers (`BodyHandler`, `HeadersHandler`, …) are **mockable traits**. Orchestrate the state
transitions in the top-level async filter, then pass the handler to plain, synchronous helper
functions that hold the logic.

**DO** — orchestrator calls testable helpers:

```rust
// State-transition orchestrator: async, owns the PDK lifecycle.
pub async fn request_filter(request_state: RequestState, config: &Config) -> Flow<()> {
    let headers_state = request_state.into_headers_state().await;
    let body_state = headers_state.into_body_state().await;

    // Hand the handler to a testable function.
    process_request(body_state.handler(), config)?;

    Flow::Continue(())
}

// Testable: takes a handler trait, no async, no PDK runtime needed.
fn process_request(handler: &dyn BodyHandler, config: &Config) -> Result<(), Error> {
    let body = handler.body();
    // business logic on `body` ...
    Ok(())
}
```

**DON'T** — logic welded to the state machine:

```rust
pub async fn request_filter(request_state: RequestState, config: &Config) -> Flow<()> {
    let headers_state = request_state.into_headers_state().await;
    let body_state = headers_state.into_body_state().await;
    let body = body_state.handler().body();

    // Business logic mixed with state transitions — can't unit test without mocking PDK.
    let processed = parse_and_validate(&body)?;
    body_state.handler().set_body(&processed)?;

    Flow::Continue(())
}
```

The payoff: `process_request` is a pure function you can drive with a mock `BodyHandler` in an
ordinary `#[test]`, while the thin async orchestrator is exercised end-to-end by [[pdk-unit-tests]].

## Important constraints (quick reference)

- **No `std::fs` or `std::net`** — wasm sandbox.
- **Use `pdk::logger`**, not `println!`/`eprintln!`; don't log in tight loops on the hot path.
- **Async only** — all I/O via PDK's async runtime.
- **No panics** — handle errors; never `unwrap()` on context-derived `Option`/`Result`. Use
  `if let` / `match` / `unwrap_or` and warn when context is absent.
- **Limited memory** — avoid large allocations; stream when possible.

## Verifying a policy against this model

- Grep the source for `unwrap(` / `expect(` / `panic!` / `println!` / `std::fs` / `std::net` —
  each is a likely violation.
- Confirm body handling streams unless the spec justifies buffering, and that any buffer has a
  documented upper bound.
- Confirm a `.local_mode()` test exists and asserts no panic with the `ApiContext` stripped.
- Confirm the spec names the fail-open/closed decision for every external dependency.
- Maintain high unit-test coverage of the business-logic helpers — they're pure functions, so
  there's no excuse not to.

## Related skills

- [[pdk-coding-best-practices]] — the concrete code patterns that satisfy these constraints.
- [[pdk-request-headers-bodies]] — streaming vs buffering bodies.
- [[pdk-unit-tests]] — `.local_mode()` and handler-driven testing.
- [[pdk-metadata]] / [[pdk-authentication]] — what's present in context, and the auth object.
- [[pdk-distributed-cache-gossip]] / [[pdk-data-storage]] / [[pdk-caching]] — cross-replica state.
- [[pdk-policy-spec]] — where degraded-mode and fail-open/closed decisions are recorded.

## Source Ref

- **Snapshot:** 2026-06-30
- **Derived from:** generalized runtime-model and constraint guidance for proxy-wasm PDK policies
  (sandbox limits, latency budget, multi-replica isolation, local-mode resilience, fail-open/closed,
  and the orchestrator/testable-helper split). Field guidance, not a single docs.mulesoft.com page.
