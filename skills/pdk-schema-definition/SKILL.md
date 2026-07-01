---
name: pdk-schema-definition
description: Use when defining configuration parameters for PDK custom policies in definition/gcl.yaml, covering metadata labels (title, description, category, interfaceScope, injectionPoint, assetTypes), property types (string, number, integer, boolean, array, object), required/optional/default values, sensitive data, conditional rendering with @visibleOn, enum and uniqueItems parameters, DataWeave expression bindings, grouping mode-specific properties into config objects, type mapping to generated config.rs, and deploy-time failures verified on the live gateway (@visibleOn+dataweave 503 crash, object-literal #[{}] default rejected, dw2pel not compiling dataweave nested in array items — expected PEL Expression).
---

# Skill: Defining a Policy Schema Definition

## Topic: Implementation

This skill covers how to define configuration parameters for PDK custom policies by editing the `definition/gcl.yaml` file. Configuration parameters appear in the API Manager UI (Connected Mode) and map to Rust types in `src/generated/config.rs`.

## GCL File Structure

The `gcl.yaml` file has two main sections:

- `metadata`: Policy information (title, description, category)
- `spec.properties`: Configuration parameters

Default template:

```yaml
apiVersion: gateway.mulesoft.com.v1alpha1
kind: Extension
metadata:
    labels:
        title: my-custom-policy
        category: Custom
spec:
    extends:
        - name: extension-definition
    properties:
        stringProperty:
        type: string
```

## Metadata Labels

```yaml
metadata:
    labels:
        title: JSON Threat Protection
        description: Protects against malicious JSON in API requests.
        category: Security
```

Supported categories: `security`, `compliance`, `transformation`, `quality of service`, `troubleshooting`.

### Interface Scope (Method & Resource Conditions)

By default, policies apply at the API level only. To enable **Method & resource conditions** in the API Manager UI (allowing users to apply the policy to specific HTTP methods and resource paths), set `metadata/interfaceScope` to `api,resource`:

```yaml
metadata:
    labels:
        title: My Policy
        category: Security
        metadata/interfaceScope: api,resource
```

- `api` — policy can be applied at the API level (default)
- `api,resource` — policy can be applied at both the API level and per resource/method

### Outbound Policies

By default, policies are inbound. To create an outbound policy:

```yaml
metadata:
    labels:
        title: My Outbound Policy
        description: An outbound security policy
        category: Security
        metadata/capabilities/injectionPoint: outbound
```

### Asset Types (instance-type targeting)

`metadata/capabilities/assetTypes` controls which Exchange instance types the
policy can attach to. It is a comma-separated list. Exchange **validates each
entry against a fixed allow-list at publish time** — an unknown value fails the
publish with `statusCode: 400, message: The asset is invalid Error in metadata:
.../capabilities/assetTypes/<i> must be equal to one of the allowed values(...)`.

Allowed values (exact spelling):

```
http, mcp, a2a, a2av1, rest, wsdl, llm, grpc, websocket, sse, graphql
```

```yaml
metadata:
    labels:
        title: REST to A2A Bridge
        category: A2A
        metadata/capabilities/assetTypes: a2a,a2av1
```

- The A2A v1 asset type is spelled **`a2av1`** — no underscore. `a2a_v1` is
  **not** a valid value and will be rejected by Exchange (a frequent mistake,
  since the protocol/spec and Rust identifiers use `a2a_v1`). Do not confuse the
  Exchange asset-type token with code-level naming.
- Omit the label entirely to attach to generic API instances only.

### Publish-time field limits

Exchange enforces length limits when the asset is published (both the definition
and, for split-model, the implementation asset). Exceeding one fails the publish
with `statusCode: 400, message: The asset is invalid request/body/<field> must
NOT have more than <n> characters` — long after a clean local build, so it only
surfaces at deploy time.

- **`description` ≤ 256 characters.** The asset description comes from
  `metadata.labels.description` in `gcl.yaml` (and, for the implementation asset,
  the `Cargo.toml` `description`). Keep it a single tight sentence or two; put the
  long-form prose in `README.md`, not the label. A multi-line YAML block (`|` /
  `>-`) still counts every character of the flattened string.
- **`title` / `name`** are likewise bounded — keep titles short.

## Define Parameters

Edit `spec.properties` in `gcl.yaml`:

```yaml
properties:
    username:
        type: string
    password:
        type: string
```

### Required Properties and Default Values

List required properties in `spec.required`. Set defaults with `default`:

```yaml
properties:
    username:
        type: string
        default: user1
    password:
        type: string
        default: 12345678
required:
    - password
```

- Required properties without a value use the default.
- Non-required properties without a value are not configured (default is only a UI suggestion).

### Sensitive Data

Protect sensitive data with `security:sensitive`:

```yaml
properties:
    key:
        type: string
        default: "YourKey"
        "@context": {
        "@characteristics": [
          "security:sensitive"
        ]
      }
```

### Conditional Parameters

Render parameters dynamically with `@visibleOn`:

```yaml
properties:
    encryptBody:
        type: boolean
        default: false
    encryptionKey:
        type: string
        "@rendering":
            "@visibleOn":
                - property: encryptBody
                  value: true
```

Restrictions:
- Single condition per parameter only
- Condition parameter must be `string`, `boolean`, or `integer`
- Conditionally rendered parameter cannot be `object` or `enum`
- Required conditional parameters must have a default value
- Only root-level parameters support conditional rendering

> **Do NOT combine `@visibleOn` with `format: dataweave` on the same property.**
> A conditionally-rendered DataWeave field is delivered to the policy
> **uncompiled** (as a raw `#[...]` string) and configuration fails on the live
> gateway with HTTP 503 — a clean local build hides it. Instead, **group the
> mode-specific DataWeave selectors into a config `object`** (see the [`object`
> type](#object) below and [Deploy-time failures](#deploy-time-failures-runtime-verified)):
> `format: dataweave` nested inside an object DOES compile, so an object cleanly
> replaces the `@visibleOn` gate without the crash.

### Enumerated Parameters

Limit input values with `enum`:

```yaml
properties:
    signingMethod:
      type: string
      enum:
        - rsa
        - hmac
```

Renders as radio buttons (3 or fewer values) or dropdown (more). Enumerated parameters don't support `@visibleOn`.

## Supported Types

### string

```yaml
properties:
    password:
        type: string
```

String format options:
- `dataweave`: Input is a DataWeave expression (Omni Gateway transforms it)
- `service`: Input is a URI; Omni Gateway auto-creates a service for HTTP calls
- `ipRange`: Input is an IP range with additional validation
- `uri`: Input is a URI with validation (use when NOT making HTTP calls)

```yaml
properties:
    externalService:
        type: string
        format: service
        default: "https://auth-service:9000/login"
    token:
        type: string
        format: dataweave
        default: "#[splitBy(attributes.headers['Authorization'], ' ')[1]]"
    blockedIps:
        type: string
        format: ipRange
        default: "192.168.3.1/30"
    loginUrl:
        type: string
        format: uri
        default: "https://anypoint.mulesoft.com/login"
```

### number

Floating point value (`f64` in Rust):

```yaml
properties:
    squareRoot:
        type: number
        default: 1.414213562
```

### integer

Integer value (`i64` in Rust):

```yaml
properties:
    maxHeaders:
        type: integer
        default: 10
```

### boolean

```yaml
properties:
    skipValidation:
        type: boolean
        default: false
```

### array

Must specify `items.type`:

```yaml
properties:
    scopes:
        type: array
        items:
            type: string
        default: ["READ", "WRITE"]
```

For a **closed value set**, constrain the items with `enum` and forbid duplicates
with `uniqueItems: true` on the array (renders as a multi-select of the allowed
values):

```yaml
properties:
    acceptedOutputModes:
        type: array
        uniqueItems: true
        items:
            type: string
            enum:
              - text/plain
              - application/json
              - image/png
```

> `uniqueItems` may be dropped from the generated `schema.json` but **survives
> into the `gcl.yaml` the gateway consumes** — the constraint is still enforced.
> **Do not put `format: dataweave` inside `items`** — the config transform does
> not compile it and the policy fails to load (see
> [Deploy-time failures](#deploy-time-failures-runtime-verified) and the
> `pdk-dataweave` skill). Use a plain-path string per item and resolve it in Rust.

### object

Must specify `properties.type` for each element:

```yaml
properties:
    user:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
      required:
        - id
        - name
      default: {"id": "1", "name": "John Doe"}
```

**Grouping objects — the preferred alternative to `@visibleOn`.** When a set of
properties applies only in one mode, group them into an object rather than
gating each with `@visibleOn` (which crashes when combined with
`format: dataweave` — see [Conditional Parameters](#conditional-parameters)).
A grouping object cleanly scopes mode-specific settings, and — verified on the
live runtime — **`format: dataweave` nested inside an object DOES forward and
compile correctly** (unlike inside array `items`, which does not):

```yaml
properties:
    continuationMode:
        type: string
        enum: [none, cache, explicit]
        default: none
    cacheConfig:                      # applies only when continuationMode = cache
        type: object
        properties:
            contextKeySelector:
                type: string
                format: dataweave     # compiles fine nested in an object
            conversationTtlSeconds:
                type: integer
                default: 3600
    explicitConfig:                   # applies only when continuationMode = explicit
        type: object
        properties:
            taskIdSelector:
                type: string
                format: dataweave
```

Rust side: model each group as its own `#[derive(Deserialize, Clone, Debug,
Default)]` struct wrapped in `Option<T>`, then flatten into your flat runtime
config with `.unwrap_or_default()` so the request hot path stays unchanged:

```rust
let cache = config.cache_config.unwrap_or_default();
// ...
context_key_selector: cache.context_key_selector,
```

## DataWeave Expression Bindings

For `dataweave` format parameters, configure bindings:

```yaml
tokenExtractor:
    type: string
    format: dataweave
    bindings:
        payload:
            mimeType: text
        attributes: true
        authentication: false
        vars: []
```

- `payload`: Request body. `mimeType`: `text`, `json`, or `xml`. Omit to restrict payload access.
- `attributes`: Request attributes (headers). Default: `true`.
- `authentication`: Authentication info. Default: `false`.
- `vars`: Variable names available to the policy. Default: `[]`. Undefined vars evaluate to `null`.

If user expressions contain bindings not defined in the schema, configuration fails.

> **`default` for an object-returning DataWeave selector must be `#[null]`, not
> `#[{}]`.** An object-literal default fails at eval with *"Object expressions
> are not yet supported"* and **silently invalidates the PolicyBinding** —
> degrading to raw passthrough with no hard error (worse than a crash, because it
> looks like it worked). Default to `#[null]` and treat a null/non-object result
> as "nothing supplied" in Rust. See
> [Deploy-time failures](#deploy-time-failures-runtime-verified).

## Type Mapping: gcl.yaml to config.rs

| gcl.yaml | config.rs |
|---|---|
| `string` | `String` |
| `uri` string | `String` |
| `ipRange` string | `String` |
| `dataweave` string | `pdk::script::Script` |
| `number` | `f64` |
| `integer` | `i64` |
| `boolean` | `bool` |
| `array` | `Vec<T>` |
| `object` | `struct` |

Non-required parameters are wrapped in `Option<T>` in Rust. Always validate before unwrapping:

```rust
if let Some(val) = my_value { /* use val */ }
```

## Propagate Changes to Rust

After modifying `gcl.yaml`, run:

```bash
# Only update config.rs
make build-asset-files

# Update config.rs AND compile the policy
make build
```

## Example: gcl.yaml to config.rs

gcl.yaml:
```yaml
properties:
    tokenExtractor:
      type: string
      format: dataweave
      default: "#[dw::core::Strings::substringAfter(attributes.headers['Authorization'], 'Bearer ')]"
    reject_missing_tokens:
      type: boolean
    max_chars:
      type: number
required:
    - tokenExtractor
```

Generated config.rs:
```rust
pub struct Config {
    #[serde(alias = "max_chars")]
    pub max_chars: Option<f64>,
    #[serde(alias = "reject_missing_tokens")]
    pub reject_missing_tokens: Option<bool>,
    #[serde(alias = "tokenExtractor")]
    pub token_extractor: pdk::script::Script,
}
```

Access in entrypoint:
```rust
#[entrypoint]
async fn configure(launcher: Launcher, Configuration(configuration): Configuration) -> Result<(), LaunchError> {
    let config: Config = serde_json::from_slice(&configuration)?;
    match config.reject_missing_tokens {
        // ...
    }
}
```

## Deploy-time failures (runtime-verified)

These constraints are **not caught by a local `make build`**. The schema
compiles, the wasm builds, tests pass — then the policy 503s or silently
degrades when the gateway *configures* it. Each was verified end-to-end against a
live Flex Gateway 1.12.1. Design the schema to avoid them up front.

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Policy fails to load, **HTTP 503** at configure time | `@visibleOn` combined with `format: dataweave` on the same property — the field reaches the policy **uncompiled** | Drop `@visibleOn`; **group the selectors into a config `object`** (dataweave nested in an object compiles) |
| 2 | Policy loads but **silently ignores the property** (raw passthrough, no error) | `default: "#[{}]"` on an object-returning selector — *"Object expressions are not yet supported"* invalidates the PolicyBinding | Use `default: "#[null]"`; treat null/non-object as "nothing supplied" in Rust |
| 3 | Policy fails to load, **HTTP 503**, `invalid type: string "#[...]", expected PEL Expression` | `format: dataweave` declared inside array `items` — the config transform (`dw2pel`) compiles only **top-level** dataweave, not array-item dataweave | Make the per-item `selector` a **plain string** and resolve the path in Rust |

**The object-vs-array asymmetry (the rule that ties #1 and #3 together).** The
gateway's `dw2pel` config transform compiles `format: dataweave` properties into
PEL expressions before the policy sees them, **but it only recurses into
top-level properties and nested *objects* — not into array *items*.** So:

- `format: dataweave` at the top level → compiles ✅
- `format: dataweave` nested inside a grouping **object** → compiles ✅ (this is
  why object grouping is the safe replacement for `@visibleOn`)
- `format: dataweave` nested inside an **array item** → NOT compiled ❌ → reaches
  the policy as a raw string → fails to deserialize into `pdk::script::Script`
  (`expected PEL Expression`) → whole policy 503s

For per-array-item selection, declare the item field as a plain `string`
(a dotted JSON path, e.g. `artifacts[0].parts[0].text`) and resolve it against
the payload in Rust. See the `pdk-dataweave` skill for the same rule from the
selector-author's side.

**Grouping objects replace `@visibleOn` (finding #1 in practice).** Rather than
gating mode-specific DataWeave fields with `@visibleOn`, put them in a per-mode
object (`cacheConfig`, `mappingConfig`, …) as shown under the
[`object` type](#object). This removes the `@visibleOn`+dataweave crash, reads
better in API Manager, and keeps the Rust hot path flat (optional group structs
flattened via `.unwrap_or_default()`).

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-create-schema-definition

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `8cafed6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-create-schema-definition.adoc`
- **Snapshot:** 2026-05-14
