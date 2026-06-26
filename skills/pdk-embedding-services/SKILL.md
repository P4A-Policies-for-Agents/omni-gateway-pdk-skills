---
name: pdk-embedding-services
description: Use when a PDK custom policy needs to compute a vector embedding for prompt text (semantic prompt-guard, semantic routing, semantic cache, classification). Covers OpenAI / Azure OpenAI / HuggingFace embedding endpoints, base64-encoded pre-computed embeddings, in-memory cosine similarity, error → HTTP-status mapping, and where to put the helper code in a custom policy crate.
---

# PDK Embedding Services

## Overview

LLM-gateway policies in this repo compute embeddings for prompt text and either (a) compare them against pre-computed exemplar embeddings in memory or (b) feed them into a vector-DB lookup. **Reuse `semantic-core::embedding_services` and `semantic-core::{cosine_similarity, calculate_topic_scores, parse_embeddings_string}` rather than rolling your own.** Only add a new helper inside `semantic-core` itself when you are onboarding a brand-new embedding provider.

Organise the helpers in a shared `semantic-core` module: provider request/response structs and the `get_<provider>_embedding` helpers together, plus the in-memory similarity helpers. `cosine_similarity` is textbook vector math:

```rust
/// Cosine similarity between two vectors. Returns -1.0 (opposite) .. 1.0 (identical).
pub fn cosine_similarity(vec_a: &[f32], vec_b: &[f32]) -> f64 {
    let dot_product: f32 = vec_a.iter().zip(vec_b.iter()).map(|(a, b)| a * b).sum();
    let magnitude_a: f32 = vec_a.iter().map(|a| a * a).sum::<f32>().sqrt();
    let magnitude_b: f32 = vec_b.iter().map(|b| b * b).sum::<f32>().sqrt();
    (dot_product / (magnitude_a * magnitude_b)) as f64
}
```

## When to Use

- New or modified policy that calls `/v1/embeddings` (OpenAI), `/embeddings?api-version=...` (Azure OpenAI), or `/pipeline/feature-extraction/...` (HuggingFace).
- Authoring a *semantic-something* policy (guard, routing, cache, dedup, similarity).
- Reviewing PRs that add an embeddings call — verify provider quirks below are honoured.
- Adding a brand-new embedding provider — extend `semantic-core::embedding_services`, do **not** create a parallel helper crate.

**Do NOT use** for generic LLM completion calls (no embeddings involved) or for non-LLM HTTP calls — see `pdk-http-call`.

## Quick Reference — Provider Quirks

| Provider     | Auth header                      | Body                                        | Path / model carrier                      | Response shape                                    |
|--------------|----------------------------------|---------------------------------------------|-------------------------------------------|---------------------------------------------------|
| `openai`     | `Authorization: Bearer <key>`    | `{ "input": "...", "model": "..." }`        | path on `Service`; model in JSON          | `{ data: [{ embedding: [f32, …] }] }`             |
| `azureopenai`| `api-key: <key>` (no `Bearer`)   | `{ "input": "..." }` — **no** `model` field | path on `Service` carries the deployment + `?api-version=` | same as OpenAI                          |
| `huggingface`| `Authorization: Bearer <key>`    | `{ "inputs": "..." }`                       | path on `Service`                         | raw `[f32, …]` — top-level array, no wrapper      |

The `provider` argument to `get_openai_embedding` is the literal string `"openai"` or `"azureopenai"`. Anything else returns `Err`.

## Iron Rules

These are checked when the policy ships. Violating any of them means the helper is wrong.

1. **The helper does not call `.path(...)` on the request builder.** The base path lives in the `Service` resource configured by the policy YAML; helpers only build headers + body and call `.post()`.
2. **Timeout comes from policy config, not a hardcoded constant.** Pass it through as `timeout_ms: u64`, then `Duration::from_millis(timeout_ms)`.
3. **Upstream failure → HTTP 503 from the policy filter, body `{"error":"Embedding service is unavailable"}`.** Not 502, not 504. The helper itself returns `anyhow::Result<Vec<f32>>`; the filter maps `Err` to a 503 via `semantic_utils::error_response(503, …)`.
4. **State between request and response filter goes through the typed `RequestData<T>` value carried by `Flow::Continue(Some(T))`.** Do not invent `set_property` / out-of-band stash mechanisms.
5. **Reuse the structs in `semantic_core::contracts`.** `OpenAIEmbeddingRequest`, `AzureOpenAIEmbeddingRequest`, `HuggingFaceEmbeddingRequest`, `OpenAIEmbeddingResponse`. Do not redefine them in a policy crate.

## Calling Pattern (Consumer)

```rust
use semantic_core::embedding_services::get_openai_embedding;
use semantic_core::semantic_utils::error_response;

let provider = config.openai_provider.as_deref().unwrap_or("openai");

let prompt_vec = match get_openai_embedding(
    &prompt_text,
    &config.openai_url,            // PDK Service
    &config.openai_api_key,
    &config.openai_embedding_model,
    &http_client,
    timeout_ms,
    provider,                      // "openai" or "azureopenai"
) .await {
    Ok(v) => v,
    Err(e) => {
        pdk::logger::warn!("[{}] embedding request failed: {}", POLICY_NAME, e);
        return Flow::Break(error_response(503, "Embedding service is unavailable"));
    }
};
```

For HuggingFace, swap `get_openai_embedding` for `get_huggingface_embedding` (no `model`, no `provider`).

## In-Memory vs Vector DB

**In-memory** (`calculate_topic_scores` + `cosine_similarity`):
- Topic exemplars are **base64-encoded little-endian f32** vectors stored in policy config (`parse_embeddings_string` → `decode_base64_embeddings`).
- Use when the exemplar set is small (≲ a few hundred vectors), shipped with the policy config, and rarely updated.
- All math is local — no second network hop.

**Vector DB** (Pinecone / Qdrant / Azure AI Search):
- Use when exemplars are large, mutable, owned by another team, or shared across policies.
- See [[pdk-vector-stores]] for the search-side rules.

Decision rule: **if exemplars fit comfortably in policy YAML and don't change between deploys, prefer in-memory.** It is cheaper, lower-latency, and has no second failure mode.

## Adding a New Provider

1. Add the request/response structs to `semantic-core/src/contracts/mod.rs`.
2. Add `pub async fn get_<provider>_embedding(...)` to `semantic-core/src/embedding_services.rs` next to the existing helpers — same signature shape (`text`, `service`, `api_key`, `…`, `http_client`, `timeout_ms`).
3. Use the PDK `HttpClient` builder. **Do not pull `reqwest` or `ureq`** — won't link in Wasm.
4. Map non-2xx to `Err(anyhow!("…API returned status {}: {}", status, body))`. Map JSON parse failures the same way. The filter's job, not yours, to translate to 503.
5. Add unit tests if there's any logic beyond serialise → call → deserialise (e.g. response unwrapping). Pure I/O helpers don't need them.

## Common Mistakes

| Mistake                                                              | Fix                                                                                  |
|----------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| Helper calls `.path("/v1/embeddings")`                                | Remove. The `Service` already points at the correct base path.                       |
| Filter returns 502 / 504 on embedding failure                         | Use 503 with `semantic_utils::error_response`.                                       |
| Azure OpenAI helper sends `Authorization: Bearer …` and a `model` JSON field | Azure uses `api-key:` header and deployment-in-URL; remove `model` from body. |
| New helper crate for a new provider                                   | Extend `semantic-core::embedding_services` instead.                                  |
| Pulling `reqwest` / `ureq` into a policy                              | Use the PDK `HttpClient` from `pdk::hl`.                                             |
| Hardcoded `Duration::from_millis(10_000)` in the helper               | Take `timeout_ms: u64` from caller; caller reads from config (`config.timeout`).     |
| Using `f32` for similarity scores end-to-end                          | `cosine_similarity` returns `f64`; thresholds in config are `f64`. Stay in `f64`.    |
| Decoding pre-computed exemplars by hand                               | `parse_embeddings_string` (JSON of base64 strings) → `decode_base64_embeddings`.     |

## Red Flags

- "I'll just write a quick `embed_with_cohere` helper directly in the policy crate" — extend `semantic-core` instead.
- "I'll set `.path("/v1/embed")` to be safe" — no. The `Service` carries the path.
- "I'll return 502 because the upstream is broken" — codebase contract is 503.
- "I'll use a trait + dyn dispatch over providers" — keep them as free async functions; the policy already knows which provider it wants.
- "I'll stash the embedding via a header / property between filters" — use `Flow::Continue(Some(T))` and the matching `RequestData::Continue` arm in `on_response`.

## Cross-References

- [[pdk-vector-stores]] — what to do with the embedding once you have it.
- [[pdk-http-call]] — generic PDK `HttpClient` patterns (timeouts, `Service`, error shapes).
- [[pdk-create-policy]] — split-model crate layout that hosts these helpers.
- [[pdk-policy-violations]] — reporting on a deny-list match in semantic guard.
