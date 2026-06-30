---
name: pdk-benchmarks
description: Use when adding, extending, or interpreting Criterion benchmarks for a PDK custom policy — measuring per-request latency, throughput, or scaling behavior. Covers bench-specific Cargo wiring (rlib crate type, [[bench]] harness=false), the pdk-unit harness reuse, the body chunk-size pitfall that silently inflates timings 50×+, the iter_batched pattern required by the process-wide proxy-wasm stub, scenario design (baseline / interesting path / scaling), and black_box / Throughput annotations.
---

# Skill: Criterion benchmarks for PDK policies

## Topic: Performance measurement

Benchmarks drive a policy through the same `pdk-unit` harness used by unit tests, so what's
measured is the real `request_filter` → header/body state-machine → upstream path the policy
executes inside Envoy. No proxy, no Wasm runtime, no Docker.

This skill documents only what's *different* for benches. Read [[pdk-unit-tests]] first — it covers
the harness conventions (Cargo deps, `enable_stop_iteration`, upstream verification) that
benchmarks share with unit tests. For the implementation patterns the policy code itself follows,
see [[pdk-coding-best-practices]].

## When to add a benchmark

A policy benefits from a Criterion benchmark when **any** of these are true:

- It runs on the request path and the spec names a latency budget.
- It buffers or transforms the body (PII rewrite, schema validation, transcoding, prompt
  decoration).
- It performs work that scales with body size, JSON depth, rule count, or token-list length.
- A change is landing that *might* regress per-request cost (regex compilation, JSON traversal,
  cache lookup).

Skip a benchmark when the policy is a thin pass-through (header check + `Flow::Continue`); the cost
is dominated by harness overhead and the numbers tell you nothing about the policy.

## Cargo wiring

Three edits to the policy's `Cargo.toml`, plus one visibility change in `lib.rs`:

```toml
[lib]
crate-type = ["cdylib", "rlib"]   # rlib is REQUIRED — benches/ can't link to a cdylib-only crate

[dev-dependencies]
pdk-unit  = { workspace = true, features = ["experimental"] }
criterion = { workspace = true }

[[bench]]
name    = "<short_name>"          # matches benches/<short_name>.rs
harness = false                   # Criterion provides its own main()
```

In `src/lib.rs`, expose the entrypoint so the bench can pass it to `with_entrypoint`:

```rust
#[entrypoint]
pub async fn configure(...) -> Result<()> { ... }
```

`#[entrypoint]` re-emits the function unchanged, so adding `pub` is harmless to the wasm build.

## File location

```
benches/<short_name>.rs
```

One file per policy is the default. Split into multiple files only when scenarios need
fundamentally different fixtures (e.g. one bench file per protocol the policy supports).

## Skeleton

```rust
use criterion::{black_box, criterion_group, criterion_main, BatchSize, BenchmarkId, Criterion, Throughput};
use pdk_unit::{TraceBackend, UnitHttpRequest, UnitHttpResponse, UnitTestBuilder};
use serde_json::{json, Value};
use std::rc::Rc;

const CONFIG: &str = r#"{"...": "..."}"#;

/// One end-to-end request through the policy. `UnitTest` cannot be
/// pre-built in `iter_batched` setup — pdk-unit's proxy-wasm host stub
/// is process-wide, and `iter_batched` instantiates a whole batch before
/// running any routine, which panics. Build the tester inline. Config
/// parsing and `configure` invocation are tiny relative to a single
/// request and are constant across iterations, so they cancel out in
/// comparisons.
fn run_request(body_bytes: Rc<Vec<u8>>) {
    let backend = Rc::new(TraceBackend::new(UnitHttpResponse::new(202)));
    let mut tester = UnitTestBuilder::default()
        .with_config(CONFIG)
        .with_backend(Rc::clone(&backend))
        .with_entrypoint(crate::configure);

    // CRITICAL — see "Chunk size" below. Without this, large-body benches are
    // dominated by per-3-byte chunk overhead, not by the policy.
    tester.set_chunk_size(1024);

    let response = tester.request(
        UnitHttpRequest::post()
            .with_path("/...")
            .with_header("content-type", "application/json")
            // black_box on the input prevents the compiler from
            // pre-folding the body; black_box on the response keeps the
            // optimizer from eliding the call entirely.
            .with_body(black_box((*body_bytes).clone())),
    );
    black_box(response.status_code());
}

fn bench_scenario(c: &mut Criterion) {
    let body = Rc::new(serde_json::to_vec(&fixture()).unwrap());
    let mut group = c.benchmark_group("<policy_name>");
    group.throughput(Throughput::Bytes(body.len() as u64));
    group.bench_function("<scenario>", |b| {
        b.iter_batched(
            // SETUP — untimed. Hand each iteration its own Rc handle to
            // the shared fixture body. Do NOT pre-build the `UnitTest`
            // here; see the comment on `run_request`.
            || Rc::clone(&body),
            // ROUTINE — timed. One full request through the policy.
            |body| run_request(body),
            BatchSize::SmallInput,
        );
    });
    group.finish();
}

criterion_group!(benches, bench_scenario);
criterion_main!(benches);
```

## The chunk-size pitfall — read this

`UnitTest`'s default body chunk size is **3 bytes**. The harness fires one `RequestBody` event per
chunk. For a 200 KB body that's ~70k events, ~70k log lines, and a benchmark dominated by harness
bookkeeping rather than the policy.

**Always call `tester.set_chunk_size(1024)` (or larger) inside the per-iteration setup** unless the
bench specifically targets streaming/chunk-boundary behavior. Concrete impact measured on a
body-transform policy:

| Body size | 3-byte chunks (default) | 1 KiB chunks | Speedup |
|---|---|---|---|
| 3.4 KB | 3.99 ms | 1.49 ms | 2.7× |
| 54 KB  | 44.4 ms | 1.98 ms | 22× |
| 210 KB | 258 ms  | ~5 ms    | 50×+ |

If a bench shows super-linear growth in body size, suspect chunk-size before suspecting the policy.

## Why `iter_batched`, not `iter`

`UnitTest` carries mutable state and isn't reusable across requests — calling `tester.request(...)`
twice would not represent a clean "one request" measurement. `iter_batched` rebuilds the tester per
iteration and **excludes that setup from timing**, so the reported number is the cost of a single
end-to-end request. `BatchSize::SmallInput` is correct for tester+body construction.

Don't `iter_with_setup` (deprecated in Criterion ≥0.5). Don't `iter` with a single tester reused
across iterations — the second call panics or produces meaningless numbers.

### Build the tester inside the timed routine (not in setup)

The standard Criterion guidance is to build the policy + tester in `iter_batched`'s untimed setup
closure and measure only `tester.request(...)`. **That does not work here.** pdk-unit's proxy-wasm
host stub is process-wide, so multiple `UnitTest` instances cannot coexist. `iter_batched`
pre-instantiates a whole batch before running any routine, which panics with `thread panicked while
processing panic. aborting.` from inside the proxy-wasm stub. Build the tester *inside* the timed
routine instead — config parsing per iteration is small (~100 µs), constant across iterations, and
cancels out in comparisons. This is the one deviation from textbook Criterion practice.

## Scenarios that pay off

A useful suite covers at least three cases, in order of importance:

1. **Clean / no-op path.** Body is well-formed, the policy walks it but does nothing. This is the
   baseline — the fixed per-request cost.
2. **The "interesting" path.** The case the policy exists for — PII rewrite, schema rejection,
   prompt decoration, rate-limit miss-then-hit. Compared against the baseline, this surfaces the
   *marginal* cost of the policy's actual job.
3. **Scaling.** A parametric input (body size, depth, rule count, token count). Use
   `bench_with_input` + `BenchmarkId::from_parameter` with `Throughput::Bytes` so Criterion reports
   both ms and MB/s. Gives you a slope, not a point.

For policies with multiple branches that are visibly different cost classes (sync allow, sync deny,
async upstream call), add a fourth scenario per branch.

## `black_box` and ownership

- `criterion::black_box(response.status_code())` (or any other return value) keeps the optimizer
  from eliding the request altogether — it can and will if you don't.
- Use `Rc<Vec<u8>>` for fixture bodies reused across iterations. Cloning the `Rc` is free; cloning a
  `Vec<u8>` adds an alloc to the hot path. The body still has to be moved into `with_body` per call
  (the API takes ownership), so dereference and clone *inside* the timed closure if needed — but
  only the bytes, not the JSON `Value`.

## Throughput annotations

When a scenario varies by body size, set `group.throughput(Throughput::Bytes(body.len() as u64))`
so Criterion reports MB/s alongside latency. Bytes-per-second is the only number comparable across
scenarios of different size, and it's what surfaces the chunk-size pitfall in the table above.

## Running benchmarks

```bash
# Compile-only smoke check (fast — does NOT run the benchmark)
cargo bench --bench <name> --no-run

# Quick run with reduced sample size (≈ 30s total). Use during development.
cargo bench --bench <name> -- --quick

# Full run with statistical confidence intervals (≈ several minutes).
cargo bench --bench <name>

# Compare against a saved baseline
cargo bench --bench <name> -- --save-baseline before
# ... make changes ...
cargo bench --bench <name> -- --baseline before
```

`--quick` and `--sample-size` are mutually exclusive; pass one or the other. Criterion writes HTML
reports to `target/criterion/` if `gnuplot` or the bundled `plotters` backend is available.

## Quieting harness logs

Even with `chunk_size=1024`, the harness emits trace lines for every dispatched event. For
development runs:

```bash
RUST_LOG=warn cargo bench --bench <name> -- --quick
```

For CI, redirect stdout to a file and grep the `time:` / `thrpt:` rows when summarizing.

## Best-practices checklist

1. **Always wrap timed inputs and outputs in `black_box`** — both the input (`run(black_box(x))`)
   and the result (`black_box(result)`), not just one side. Otherwise the optimizer can constant-fold
   the whole iteration.
2. **Build the tester inside the timed routine** (see above) — the process-wide stub forbids
   pre-building in setup; config parsing is cheap and cancels out.
3. **Benchmark realistic scenarios** — real method names, plausible payload and config shapes.
   Synthetic micro-payloads under-report cost because they skip the JSON-walking and regex sweeps
   that dominate real workloads.
4. **Use `BenchmarkId` for parametric scenarios** — `bench_with_input` + `BenchmarkId::from_parameter`
   paired with `Throughput::Bytes(...)`. You get a *slope*, which tells you whether the code is O(n),
   O(n log n), or accidentally O(n²).
5. **Compare alternatives in the same group** — add each as a separate `bench_function` in one
   `benchmark_group`; the HTML report shows them side-by-side with confidence intervals.
6. **Cover success AND failure paths when their cost differs** — `Flow::Continue` cheap vs
   `Flow::Break` expensive, cache hit vs miss, validated vs rejected. Production average depends on
   the failure rate.
7. **Use multiple data sizes — small, medium, large.** A single point is a fact; three points are a
   function. Small bodies surface fixed overhead, large bodies surface scaling.

## What benchmarks are NOT

- **Not a substitute for unit tests.** Benchmarks measure cost; tests verify correctness. Add the
  unit test first ([[pdk-unit-tests]]); a policy without tests should not be benchmarked — Criterion
  happily times the wrong behavior.
- **Not a regression gate by default.** Criterion can fail-on-regression with `--baseline`, but
  wiring that into CI is a separate decision; default benches are advisory.
- **Not for cross-replica behavior.** Each iteration runs in a single in-process tester; nothing
  here exercises multi-replica state, distributed caches, or real network latency. For that, use
  Docker-based integration tests ([[pdk-integration-tests]]).

## Related skills

- [[pdk-unit-tests]] — the harness benchmarks reuse; read first.
- [[pdk-coding-best-practices]] — the policy patterns being measured.
- [[pdk-runtime-model]] — why latency is a product constraint and the streaming/buffering rules
  that benchmarks exist to validate.
- [[pdk-distributed-cache-gossip]] — for cost questions that cross replicas (out of bench scope).

## Source Ref

- **Snapshot:** 2026-06-30
- **Derived from:** hands-on Criterion benchmarking of PDK body-transform policies through the
  `pdk-unit` in-process harness; the chunk-size pitfall and the inline-tester-construction caveat
  are generalized field findings, not from a docs.mulesoft.com page.
