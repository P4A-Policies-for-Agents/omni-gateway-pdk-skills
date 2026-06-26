---
name: pdk-vector-stores
description: Use when a PDK custom policy queries a vector database (Pinecone, Qdrant, Azure AI Search) for top-K nearest-neighbour search of an embedded prompt — semantic prompt-guard, semantic routing, semantic cache. Covers the supported backends and their REST quirks, the shared VectorSearchResult contract, threshold + deny-list filtering, the Distance-Weighted KNN routing algorithm, and the telemetry headers consumer policies must emit.
---

# PDK Vector Stores

## Overview

Once a policy has a `Vec<f32>` embedding (see [[pdk-embedding-services]]), it queries a vector database for top-K nearest neighbours. **Three backends are supported and ALL of them live in `semantic-core::vector_db`: Pinecone, Qdrant, Azure AI Search.** Reuse `search_pinecone` / `search_qdrant` / `search_azure_ai_search` and the shared `VectorSearchResult` contract — do not introduce a new backend abstraction or a new HTTP client.

The three backends live in a shared `semantic-core::vector_db` module; reuse `search_pinecone` / `search_qdrant` / `search_azure_ai_search` and the shared `VectorSearchResult` contract (shown below) rather than introducing a new backend abstraction or HTTP client.

## When to Use

- Authoring or editing a `semantic-prompt-guard-policy-*` or `semantic-routing-policy-*`.
- Onboarding a new embedding/vector-DB combination (a new policy crate; the search helper itself is reused).
- Adding a brand-new vector-DB backend — extend `semantic-core::vector_db`, do not start a parallel module.
- Reviewing a PR that queries a vector store from a PDK policy.

**Do NOT use** for raw HTTP calls unrelated to vector search (see [[pdk-http-call]]) or for the embedding step itself (see [[pdk-embedding-services]]).

## Shared Contract

Every backend returns the same shape so the policy filter is backend-agnostic:

```rust
pub struct VectorSearchResult {
    pub topic:   String,
    pub content: Option<String>, // optional document text for prompts/observability
    pub score:   f64,            // higher = more similar
}
```

Stored documents must carry `topic` and (optionally) `text` payload. The backend helpers map their native shape onto this — that mapping is the *only* place backend differences live.

## Backend Quirks (Reference)

| Backend         | Path                                                    | Auth header              | Required extra headers / params                       | Where `topic`/`text` live                  | Score field            |
|-----------------|---------------------------------------------------------|--------------------------|-------------------------------------------------------|--------------------------------------------|------------------------|
| Pinecone        | `POST /query`                                           | `Api-Key: <key>`         | `X-Pinecone-Api-Version: 2026-04` (date-versioned)    | `metadata.topic`, `metadata.text`          | `score`                |
| Qdrant          | `POST /collections/{collection}/points/search`          | `api-key: <key>`         | —                                                     | `payload.topic`, `payload.text`            | `score`                |
| Azure AI Search | `POST /indexes/{index}/docs/search?api-version=2025-09-01` | `api-key: <key>`     | `vectorQueries[0].kind="vector"`, `fields="contentVector"` | top-level `topic`, `text`            | `@search.score`        |

Pinecone uses `topK` + `namespace`; Qdrant uses `limit` + collection-in-path; Azure uses `k` + `vectorQueries[].fields` (the indexed vector field name is hardcoded `contentVector` — change requires re-indexing AND a code change here).

## Iron Rules

1. **All backend helpers are free async functions.** No `trait VectorBackend`, no `Box<dyn …>`, no `async_trait`. The policy crate selects which helper to call based on its own config and which backend its YAML wires up.
2. **Threshold filtering is client-side, not pushed into the query.** Always request top-K, then filter `score >= threshold` in Rust. Backends report scores in inconsistent ways (cosine score vs certainty vs distance) — keep the comparison in one place.
3. **Failure → HTTP 503 from the policy filter, body `{"error":"Vector search service is unavailable"}`.** Same pattern as embeddings (see [[pdk-embedding-services]]). Use `semantic_utils::error_response(503, …)`.
4. **Gate the request before calling search.** Use `semantic_utils::should_process_request(input_format, method, content_type)` — semantic policies only run on OpenAI-format JSON POSTs.
5. **Routing scoring is Distance-Weighted KNN, not max / mean / top2-mean.** Use `semantic_core::vector_db::find_best_route_from_search`. Do not invent a new aggregation.
6. **Telemetry headers are exact strings.** `x-llm-proxy-semantic-guard-success` and `x-llm-proxy-semantic-routing-success` — emit via `SemanticGuardOutcome::header_value()` and `RoutingDecision::header_value()` from `semantic_core::contracts`.

## Distance-Weighted KNN (Routing)

`find_best_route_from_search(search_results: &[VectorSearchResult], routes: &[&RouteTopicMeta], threshold: f64) -> Option<RouteMatch>`:

1. Drop hits with `score < threshold`.
2. Group surviving hits by `topic`; per-topic score = arithmetic mean of those hit scores.
3. For each route, route score = **max** over its topic averages.
4. Best route = route with highest route score (ties → first in iteration order). Returns `Some(RouteMatch { provider, model, score, topic })` or `None`.

`routes` is a slice of **references** (`&[&RouteTopicMeta]`) so the caller can pass a `.filter().collect()` view of header-eligible routes without cloning — header eligibility is the caller's responsibility; this function does not consult `routes[].headers`.

Worked example (from the unit tests): one route, topics `["Code","Debug"]`. Hits: `Code 0.85, Code 0.80, Code 0.75, Debug 0.88`. Then `avg(Code)=0.80`, `avg(Debug)=0.88`, route score = `max(0.80, 0.88) = 0.88`, best topic = `Debug`.

Why this and not max-only or mean-only: a single fluke high-similarity exemplar shouldn't crown a topic, but a topic with many on-target exemplars also shouldn't be diluted by one outlier — averaging *within* a topic and taking the *max across* a route's topics balances both.

## Filter Pattern (Semantic Guard)

```rust
// 1. Gate
if !should_process_request(handler.header(X_INPUT_FORMAT_HEADER),
                           headers_state.method().as_str(),
                           handler.header("content-type").as_deref()) {
    return Flow::Continue(None);
}

// 2. Embed (see pdk-embedding-services)
let prompt_vec = match get_openai_embedding(...).await { ... };

// 3. Search top-K
let search_results = match search_pinecone(
    &prompt_vec, &cfg.pinecone_url, &cfg.pinecone_api_key,
    cfg.pinecone_namespace.as_deref().unwrap_or("default"),
    PINECONE_SEARCH_LIMIT, &http_client, timeout,
).await {
    Ok(r) => r,
    Err(e) => return Flow::Break(error_response(503, "Vector search service is unavailable")),
};

// 4. Filter client-side and pick highest-scoring deny match
let highest_deny = search_results.iter()
    .filter(|r| r.score >= cfg.threshold && deny_topic_names.contains(&r.topic))
    .max_by(|a, b| a.score.partial_cmp(&b.score).unwrap());

if let Some(r) = highest_deny {
    policy_violations.generate_policy_violation();
    return Flow::Break(Response::new(403)
        .with_headers(vec![(CONTENT_TYPE_HEADER.to_string(), APPLICATION_JSON.to_string())])
        .with_body(format!(
            r#"{{"error":"Request denied - Prompt semantically matches deny list","topic":"{}","score":{:.3}}}"#,
            r.topic, r.score)));
}

// 5. Continue with telemetry; emit header in on_response
let outcome = match search_results.first() {
    Some(r) => SemanticGuardOutcome::Passed { topic: Some(r.topic.clone()), score: r.score },
    None    => SemanticGuardOutcome::Passed { topic: None, score: 0.0 },
};
Flow::Continue(Some(outcome))
```

## Filter Pattern (Semantic Routing)

After search, build an `&[&RouteTopicMeta]` of header-eligible routes and call `find_best_route_from_search(&hits, &eligible_routes, threshold)`. On `Some(RouteMatch { provider, model, score, topic })` rewrite the body to target that route's provider/model and stash a `RoutingDecision { provider, model, is_fallback: false, best_topic: Some(topic), score }`. On `None`, fall back to the configured default route with `is_fallback: true`. Emit the `X_LLM_PROXY_SEMANTIC_ROUTING_SUCCESS` header from `RoutingDecision::header_value()` in the response filter. See `semantic_core::semantic_utils::build_routing_outcome`.

## Adding a New Backend

1. Create `semantic-core/src/vector_db/<backend>.rs` with `pub async fn search_<backend>(embedding, service, api_key, …, http_client, timeout_ms) -> Result<Vec<VectorSearchResult>>`.
2. Map the backend's response into `VectorSearchResult { topic, content, score }`. Drop entries missing a `topic`.
3. Use the PDK `HttpClient`. No `reqwest` / `ureq`.
4. Status non-2xx → `Err(anyhow!("…returned status {}: {}", …))`. Filter maps to 503.
5. Add `pub mod <backend>;` to `vector_db/mod.rs`.
6. Add `cargo test` cases for any score-extraction edge cases (missing payload, missing topic).

## Common Mistakes

| Mistake                                                                | Fix                                                                       |
|------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Inventing a `VectorBackend` trait or `dyn` abstraction                  | Use free `search_*` functions; the policy crate picks one.               |
| Returning `Hit { id, score: f32 }` or any custom shape                  | Use `semantic_core::vector_db::VectorSearchResult` (`f64` score).        |
| Pushing the threshold into the search request                          | Always request top-K; filter `score >= threshold` client-side.            |
| Picking Weaviate / Milvus / pgvector / Chroma                           | Not supported. Pinecone / Qdrant / Azure AI Search only.                  |
| Forgetting `X-Pinecone-Api-Version` header                              | Required for Pinecone data-plane queries — date-versioned.                |
| Forgetting Azure `?api-version=2025-09-01` query param                  | Required.                                                                 |
| Hardcoding the Azure vector field name in policy config                 | It's `contentVector` in code; document this for the indexer team.         |
| Inventing a routing aggregation (`max`, `mean`, `top2_mean`)            | Use `find_best_route_from_search` — Distance-Weighted KNN.                |
| Skipping `should_process_request`                                       | Semantic policies must only run on OpenAI-format JSON POSTs.              |
| Hand-formatting the telemetry header                                    | Use `SemanticGuardOutcome::header_value` / `RoutingDecision::header_value`. |
| Returning 502 / 500 on backend failure                                  | 503 with `error_response(503, "Vector search service is unavailable")`.   |

## Red Flags

- "I'll add Weaviate support" — not in scope; ask first. Three backends are intentional.
- "I'll abstract the three backends behind a `VectorBackend` trait so the policy can pick at runtime" — no; the policy crate is per-backend by design (one crate per backend × embedding-provider combination).
- "I'll just push the threshold into the query for efficiency" — no; backend score semantics differ, keep filtering local.
- "I'll write `format!("topic={};score={:.3}", …)` for the success header" — use `header_value()`; the prose is exact.
- "Routing should pick the highest single hit" — that's max-only and is not the algorithm in this repo. Use `find_best_route_from_search`.

## Cross-References

- [[pdk-embedding-services]] — produces the `Vec<f32>` this skill consumes.
- [[pdk-http-call]] — `HttpClient` patterns shared by all backend helpers.
- [[pdk-policy-violations]] — required when blocking on a deny-list match.
- [[pdk-create-policy]] — split-model crate layout for new `semantic-*` policies.
- [[pdk-schema-definition]] — defining `threshold`, `deny_topics`, `route_topics`, backend creds in policy YAML.
