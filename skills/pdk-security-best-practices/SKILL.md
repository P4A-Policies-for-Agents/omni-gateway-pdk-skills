---
name: pdk-security-best-practices
description: Use when writing or reviewing any PDK policy that touches credentials, API keys, tokens, secrets in gcl.yaml, authentication or authorization decisions, client IPs, injected backend headers, outbound HTTP calls, or AI prompt/PII content, and before shipping a policy, to check its security posture (leaked secrets, timing attacks, spoofed X-Forwarded-For, SSRF, error oracles, fail-open vs fail-closed, DoS limits).
---

# Skill: Security Best Practices for PDK Policies

## Topic: Implementation (cross-cutting)

A policy runs **on the credential path** of every request. It holds operator secrets, sees caller
credentials, and its decisions are the access control. Treat it like this:

- **Caller input is hostile:** every header (including `X-Forwarded-*`), the path, the query, the
  body and the token claims.
- **Operator config is trusted but secret:** its values must never reach a log, an error, a
  response, a metric or a span.
- **Upstream responses are untrusted:** the backend, the IdP and the LLM can all return redirects,
  huge bodies, or error bodies that echo secrets.

MuleSoft's own policies don't apply all of this consistently, and some public PDK examples
contradict it (see the anti-patterns below). Follow this skill, not the nearest example.

## 1. Secrets in configuration (`gcl.yaml`)

Mark every secret property with the `security:sensitive` characteristic. The syntax is exact. This
characteristic is the documented marker; a guessed `sensitive: true` or `format: secret` is not it,
and the value is then handled as plain config:

```yaml
properties:
  hmacKey:
    type: string
    minLength: 1            # an empty key is a public key: reject it at apply time
    "@context":
      "@characteristics":
        - "security:sensitive"
```

- Mark each secret: API keys, client secrets, passwords, HMAC or signing keys, private keys and
  static tokens. Arrays of keys need the marker too.
- Don't give a secret a `default`. Don't commit real secrets in `playground/config/*.yaml` or in
  test fixtures; use obviously fake values.
- Keep secrets out of `{:?}`. Give any type that holds a secret a hand-written `Debug` (and
  `Display`) that prints `<redacted>`, and unit-test that it does:

  ```rust
  impl fmt::Debug for SigningKey {
      fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { f.write_str("SigningKey(<redacted>)") }
  }
  impl fmt::Debug for TokenResponse {
      fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
          f.debug_struct("TokenResponse").field("access_token", &"<redacted>")
              .field("expires_in", &self.expires_in).finish()
      }
  }
  ```
- `src/generated/config.rs` is generated, so you can't give `Config` a custom `Debug`. Never
  format the whole `Config` (`{:?}`). If you keep secrets in your own runtime struct, wrap them in
  a newtype with a redacting `Debug`.
- On a config-parse or validation error, return `Err` from `configure` (that is the fail-closed
  route: the policy doesn't start) and log the error only, never the payload:
  `"invalid configuration: {e} (configuration payload redacted)"`.

See [[pdk-schema-definition]] for the rest of the schema.

## 2. Logs, errors, metrics and spans

The full rules are in [[pdk-policy-logging]]. In short:

- Never log a credential, a header dump or a body. Log presence or length only; a hash or prefix
  still counts as the credential.
- The principal can be the raw token. Log `client_id`.
- Error variants carry a *kind*, not the value. Detected PII is reported as type and position only.
- **Log flooding is an attack surface.** Each rejected request is attacker-triggered, so log
  rejections, and failures of side calls made *on the rejection path*, at `debug!`. Keep `warn!`
  for operator-actionable problems that a caller can't trigger at will (misconfiguration, a
  dependency outage on the success path). If you need to surface a dependency outage, warn only
  on the state change (track a flag), not per request.
- **Client IPs are personal data, but operational.** Logging the connection peer IP at `debug!` or
  sending it to a configured audit service is fine. Never put it in a metric or span tag.
- Metric tags and span tags must be bounded and must never carry caller identity or content
  ([[pdk-coding-best-practices]], "Bounded metric tags"). A raw client IP or token as a tag is both
  a leak and a cardinality bomb.

## 3. Comparing secrets: constant time

Never compare a caller-supplied secret (API key, password, HMAC signature) with `==`. It returns on
the first mismatching byte, which lets an attacker guess the secret byte by byte from response
timing. Use the `subtle` crate (`subtle = "2"`, pure Rust, builds for `wasm32-wasip1`):

```rust
use subtle::ConstantTimeEq;

// A length mismatch returns early, which reveals only the length, not the secret.
fn ct_eq_str(a: &str, b: &str) -> bool { a.as_bytes().ct_eq(b.as_bytes()).into() }

// Several valid keys: check all of them, don't stop at the first match.
fn key_is_valid(candidate: &str, keys: &[String]) -> bool {
    !candidate.is_empty() && keys.iter().fold(false, |ok, k| ok | ct_eq_str(candidate, k))
}
```

For HMAC request signing, also bound the replay window with a timestamp tolerance, and include the
timestamp in the signed string.

## 4. Authentication and authorization failures

- **One fixed body for every failure step.** Don't return "missing header", "failed to decode",
  "failed to decrypt" or "wrong key" as different bodies; each one tells an attacker which step
  their input got past. Put the detailed reason in a `debug!` line or a policy violation
  ([[pdk-policy-violations]]).
- **The right status.** 401 for missing or invalid credentials (with a `WWW-Authenticate`
  challenge), 403 for a valid identity that isn't allowed, 400 or 403 for content blocks (PII,
  guardrail), never 401. Bad credentials must never surface as a 5xx; for example, an IdP answering
  `400` means the token is invalid ([[pdk-token-introspection]]).
- **Never echo input.** Don't put the token, the principal, the matched PII or the IdP's error body
  in the response. A deny body may name the *resource*, never the caller's credential.
- Add `cache-control: no-store` to auth error responses.

## 5. Caller-controlled request data

- **`X-Forwarded-For` is spoofable.** Its leftmost entry is whatever the caller wrote. Never use it
  for an allow/deny list, a rate-limit key or an audit record unless the gateway sits behind
  proxies you control. If it does, take the entry `trusted_hops` positions from the **right**
  instead. Without trusted proxies, use the connection peer ([[pdk-share-data-between-policies]]).
  Its value is `ip:port`, so strip the port before comparing:

  ```rust
  let peer = stream.read_property(&["source", "address"])
      .and_then(|b| String::from_utf8(b).ok())
      .map(|a| a.rsplit_once(':').map_or(a.clone(), |(ip, _)| ip.trim_matches(['[', ']']).to_string()));
  ```
- **`X-Forwarded-Proto`/`X-Forwarded-Host`** are fine for building a URL you *advertise* (RFC 9728
  metadata), but don't use them for a security decision (host-based routing or allowlists).
- **Strip before you inject.** Before injecting a backend credential, remove the caller's copy of
  the target header and any operator-listed headers (case-insensitively), and inject credential
  pairs as a pair. Then also remove the credential the *caller* authenticated with (`x-api-key`,
  `Authorization`) so it doesn't reach the backend. The full pattern is in
  [[pdk-request-headers-bodies]].
- **Strip your internal headers.** If your policies communicate through a header namespace
  (`x-myorg-*`), remove that namespace from inbound requests at the first policy, or a caller can
  forge it. Prefer stream properties, which callers can't set ([[pdk-share-data-between-policies]]).
- **Header names built from data** (for example `x-claim-<name>` from JWT claims) must be validated
  as an RFC 9110 token (`[!#$%&'*+.^_`|~0-9A-Za-z-]+`). Skip the header if the name fails.

## 6. Outbound calls

- **No SSRF.** Only call a host the operator configured (`format: service` /
  [[pdk-http-call]]). Never build the target host or URL from request data.
- **Refuse upstream redirects** in transcoders and proxies to tools: a 3xx from a backend becomes an
  error and a policy violation, not a followed redirect ([[pdk-mcp]]).
- **Set a timeout on every call.** Range-check configurable timeouts (for example 1000 to 30000 ms).
- **Never forward the caller's `Authorization`** to a third-party service (guardrail, audit,
  moderation). Authenticate with the policy's own credential.
- **Fail closed when the outbound credential is missing.** If a signing or credential-injection
  policy has no key, return a generic 500 ("credentials are not configured; request not signed").
  Never send the request unsigned. Overwrite any inbound `Authorization` before signing.
- **Validate what the IdP hands back.** Reject a token response whose `token_type` isn't `Bearer`,
  never cache an empty `access_token`, and truncate any IdP error body you log.
- **Calls on the rejection path cost you.** A call made for every *rejected* request (audit,
  alerting) is a request flood the attacker controls. Keep the timeout short (hundreds of ms), fail open,
  and log its failures at `debug!`. Where a policy violation is enough
  ([[pdk-policy-violations]]), prefer it to a per-rejection HTTP call.

## 7. Resource limits (DoS)

- Name every cap as a constant, and check it before allocating: the body size, the XML/JSON depth,
  the certificate size, the number of tools, the length of a free-text field, the SSE bytes pending.
- Use the PDK JSON/XML validators' `maxDepth`, `maxArrayLength` and `maxStringLength`
  ([[pdk-json-validator]], [[pdk-xml-validator]]).
- Stream rather than buffer when the body can be large ([[pdk-sse-parsing]]).
- No `.unwrap()`/`.expect()`/indexing on caller or upstream data; a panic is a remotely triggered
  outage. Fuzz or property-test your parsers and sanitizers.
- The `regex` crate is linear time, but compile caller-*influenced* patterns only from config, with
  `RegexBuilder::size_limit`, at `configure` time.
- Rebuild instead of forwarding: if callers can send structured input to an upstream (GraphQL
  selections, filters), parse it, allow a minimal subset, and re-serialize it from the AST. Never
  pass the raw string through.

## 8. Fail closed or fail open

| Concern | Default | Notes |
|---|---|---|
| Authentication, signature verification, credential injection | **Closed** | A missing key or an IdP failure denies the request |
| PII detection, content guardrail | **Closed** (configurable `failOpen`, default `false`) | When closed, always return the same status (for example 503); only a reason header varies |
| Rate limiting on an infra glitch (storage unreachable) | Open, configurable | Availability over strictness; refuse to *start* on a misconfiguration |
| Telemetry, metrics, audit side channels | **Open** | Never affects the response |
| Policy config that fails to parse or evaluate | **Closed** | Fail `configure`; never run in a degraded "allow all" mode |

Make the choice explicit: a config flag, or a comment saying why. A silent `Flow::Continue(())` in
an error branch of a security policy is a bypass.

## 9. AI traffic (LLM, MCP, A2A)

- **Content capture is opt-in.** Prompts, completions, tool arguments and tool definitions go into
  logs, events or spans only when a `captureContent`-style flag is set, and it defaults to `false`.
- **Redaction and transformation are opt-in**, and they're applied only when they map exactly onto
  the original content. Otherwise, treat the result as undetermined and fail closed.
- **Never echo detected PII** in a block response.
- Free-text fields that an agent controls (justifications, descriptions) are content. Log their
  length, not their text.

## Anti-patterns found in the public PDK examples

These examples are meant to teach the API. Don't copy their security posture:

| Example (`mulesoft/pdk-custom-policy-examples`) | Problem | Do instead |
|---|---|---|
| `jwt-validation/definition/gcl.yaml` | `secret` without `security:sensitive`; a real-looking secret in the playground config | Mark it sensitive and use a fake value (section 1) |
| `crypto/src/lib.rs` | Logs the decrypted nonce at `debug!`; a different 401 body for each failure step | Never log secrets; use one fixed body (section 4) |
| `agent-policies/a2a-pii-detector-policy/src/lib.rs` | Returns the detected PII in the response body, with a 401; `.unwrap()` on the detector output | Report type and position only, with 400/403; no unwrap |
| `metrics/src/lib.rs` | Logs the full outbound body at `debug!` | Log the length |

`simple-oauth-2-validation-grpc` is a good model: a bare 401 with `WWW-Authenticate: Bearer
realm="oauth2"`, and the details at `debug!` only.

## Pre-ship checklist

- [ ] Every secret in `gcl.yaml` has `security:sensitive`, no default, and `minLength: 1` where empty
      is meaningless.
- [ ] Grep the `debug!|info!|warn!|error!|format!|anyhow!` calls: no header values, tokens,
      principals, bodies, config bytes or secret-bearing `{:?}`.
- [ ] Types holding secrets have redacting `Debug`/`Display`, with a test.
- [ ] Secrets are compared with `subtle::ConstantTimeEq`.
- [ ] Every auth failure path returns the same fixed body; the status is 401/403, never 5xx for bad
      credentials.
- [ ] No security decision uses the leftmost `X-Forwarded-For` or any caller header you didn't
      strip first.
- [ ] Outbound hosts come from config only; redirects are refused; every call has a timeout.
- [ ] Caps are constants and checked; no unwrap on input.
- [ ] Each error branch is a deliberate fail-closed or fail-open choice.
- [ ] Rejections log at `debug!`.

## Source Ref

- **Sensitive parameters:** https://docs.mulesoft.com/pdk/latest/policies-pdk-create-schema-definition
  (`security:sensitive`), checked 2026-11-11.
- **Public examples cited:** `mulesoft/pdk-custom-policy-examples` (`jwt-validation`, `crypto`,
  `agent-policies/a2a-pii-detector-policy`, `metrics`, `simple-oauth-2-validation-grpc`,
  `json-validation`), checked 2026-11-11.
- **Field-derived (2026-11-11):** the redacting `Debug`, the constant-time comparison, the
  strip-then-inject, the redirect refusal, missing-credential fail-closed, opt-in content capture
  and the fail-mode table are drawn from MuleSoft's own production policies and their security
  fixes. Some of MuleSoft's own built-in policies still compare passwords with `==` or trust the
  leftmost `X-Forwarded-For`; this skill deliberately recommends against both.
- `subtle`: https://docs.rs/subtle
