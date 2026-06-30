---
name: pdk-policy-spec
description: Use when reading, writing, or updating a PDK policy or library spec (docs/spec.md) — drafting a spec for a new policy before code (specification-first development), updating a spec after a behavior change, or reviewing one for drift against the code. Covers the one-spec-per-crate discipline, the policy vs library spec shapes, the fixed section order, what does NOT belong in a spec, and the keep-in-sync rules. Ships policy-template.md and library-template.md.
---

# Skill: PDK policy & library specifications

## Topic: Documentation discipline

A spec (`docs/spec.md`) is the single source of truth for a policy or library: what it does, how
it's configured, how it behaves. It's internals-facing — for contributors, reviewers, and AI
agents — not the consumer-facing Exchange listing. Keep it next to the code and keep it in sync:
a behavior-changing edit updates the spec in the same commit. A spec that lags the code is worse
than no spec.

Two shapes, picked by what the crate is:

- **Policy spec** — anything that compiles to WebAssembly and runs in the gateway. See
  [policy-template.md](policy-template.md).
- **Library spec** — shared Rust crates that policies depend on. See
  [library-template.md](library-template.md).

Both follow the same discipline: a single H1, fixed section order, no extra top-level sections, one
file living beside the code.

## When this skill applies

- **Drafting a new policy or library** — write `spec.md` *before* the code (see "Process" below).
- **Answering a question** about an existing policy's configuration, behavior, or architecture —
  read its spec first.
- **After a behavior/config/API change** — update the spec in the same change.
- **Reviewing for drift** — checking that the spec still matches what the code does.

## Policy spec shape

```
# <Policy Name>

<one-paragraph summary>

## Configuration
## Behavior
## Architecture
## Examples
```

### H1 — policy name
Exactly one H1, matching the policy's display name (e.g. `MCP Schema Validation Policy`). Follow it
with a one-paragraph summary: what it does and why it exists. No bullet lists, no tables — one
paragraph. This is the quick pitch.

An optional **Category** line immediately after the H1 is useful when your project classifies
policies (e.g. by protocol or concern). If you use one, the value must match whatever taxonomy the
project's policy definitions use — it's a project convention, not a fixed PDK requirement. Omit it
if your project doesn't categorize policies.

### Configuration
One `### <propertyName>` subsection per top-level property in the policy's definition
([[pdk-schema-definition]]). Each gives: type, required/optional, default; what it does in a
sentence or two; a concrete example value; and any gotchas **inline**. Headings use the property
name **verbatim** so they're checkable against the config schema. Omit the whole section if the
policy has no configuration.

### Behavior
The observable contract — usually the largest and most-read section:
- **Request flow** — what it inspects, what it mutates, when it breaks the chain.
- **Response flow** — only if the policy touches responses.
- **Rules** — **numbered** when the policy enforces checks, so tests and error messages can
  reference them.
- **Missing context (local-mode)** — what happens with no control-plane connection. Document the
  degraded behavior per the runtime model ([[pdk-runtime-model]]): default is warn-and-continue;
  any stricter mode is an explicit, justified override. The authentication object is a filter-chain
  concern, not a local-mode one — note its absence under Edge cases if relevant, not here.
- **Edge cases** — as bullets, not a separate top-level section.
- **Error responses** — conditions with their status codes (and protocol error codes, e.g.
  JSON-RPC, where applicable).
- **Benchmarks** — a `### Benchmarks` subsection naming the latency budget and the scenarios a
  Criterion bench covers ([[pdk-benchmarks]]). Skip only for a thin pass-through whose cost is
  harness-dominated.

### Architecture
Internals only — components, data flow, extension points, non-obvious dependencies. Include it
*only* when someone changing the code would benefit from a component map. Skip for single-filter
policies.

### Examples
Complete, runnable config + request/response samples for the common cases. At minimum one
happy-path example. Fenced blocks with correct language tags (`yaml`, `json`, `rust`, `http`).

## Library spec shape

```
# <Lib Name>

<one-paragraph summary>

## What it provides
## Examples
```

- **H1** matches the crate name (e.g. `token_cache`), followed by a one-paragraph summary.
- **What it provides** — the contract: the problem it solves, the mental model, the load-bearing
  public types/functions (in backticks), and — crucially — what it deliberately does **not** solve.
  Don't reproduce rustdoc; `cargo doc` is the authoritative API reference, this section is the
  narrative.
- **Examples** — short Rust snippets (3–15 lines each), one per common use case, tagged `rust`.
  Policies are the integration tests; don't duplicate them here.

## What NOT to write (both shapes)

These are **not** top-level sections in any spec — fold the info into the section it belongs to:

- ~~Performance Considerations~~ → Behavior or Architecture (policy) / What it provides (lib).
- ~~Security Considerations~~ → same.
- ~~Logging~~ → same; document log lines only if they're part of the public contract.
- ~~Troubleshooting~~ → inline under the relevant property, rule, or capability.
- ~~Edge Cases~~ → inside Behavior as bullets.
- ~~Compatibility~~ → short bullets at the end of Behavior / What it provides if non-trivial, else
  skip.
- ~~Quick Reference~~ → the spec itself is the reference.
- ~~Dependencies~~ → `Cargo.toml` is the source of truth.
- ~~Test Coverage~~ → CI owns that.
- ~~API reference~~ (lib) → `cargo doc` enumerates; the spec narrates.

"Needs its own H2" is almost never the right answer — find the section the info most belongs to.

## Process

1. **New policy or lib** — write `spec.md` **before** writing code. A spec-first review catches
   design problems earlier and cheaper than tests ever will. A good rhythm is to land the spec for
   review first (e.g. its own PR), align on the design, then implement against the approved spec.
2. **Existing code without a spec** — write it from the current code; use the exercise to surface
   missing behavior or accidental complexity.
3. **Behavior / config / public-API change** — update `spec.md` in the same commit.
4. **Pure refactor** — no spec change needed.

## Writing tips

- **Be specific**: "returns HTTP 403" beats "denies the request"; "JSON-RPC error -32602" beats
  "returns an error".
- **Avoid marketing tone**: "intercepts `tools/call` and validates arguments against the tool's
  `inputSchema`" beats "provides comprehensive schema validation".
- **Quote names literally** in backticks — property, method, type, header names.
- **No end-of-section summaries** — every paragraph should add new information.
- **Tables for enumerated values**, not for prose.
- **Keep examples minimal** — just enough to exercise the feature.

## Checking a spec for drift

**Policy spec:**
- Every top-level property in the definition has a matching `### <propertyName>` subsection, and
  every subsection maps to a property that still exists.
- Error/status codes in Behavior match what the code emits (`grep` the error constants in source).
- If a `### Benchmarks` subsection names a bench file, that file exists and its named scenarios
  match the `bench_function` / `bench_with_input` calls in it ([[pdk-benchmarks]]).
- The Category line (if present) matches the project's taxonomy for that policy.

**Library spec:**
- Every public type / function / trait the spec mentions still exists.
- Every example still compiles in your head — spot-check imports and type names against source.

**Both:**
- The summary paragraph still matches what the code does (most common drift: a feature was added but
  the summary still describes the smaller, earlier version).

## Related skills

- [[pdk-create-policy]] — scaffolding the policy the spec describes.
- [[pdk-schema-definition]] — the `gcl.yaml` properties the Configuration section documents.
- [[pdk-runtime-model]] — the runtime facts behind the Missing-context and fail-open/closed content.
- [[pdk-benchmarks]] — the Criterion benches the Behavior section's Benchmarks subsection names.

## Source Ref

- **Snapshot:** 2026-06-30
- **Derived from:** generalized specification-first documentation discipline for PDK policies and
  libraries (one spec per crate, fixed section order, keep-in-sync rules). Project-process guidance,
  not a docs.mulesoft.com page.
