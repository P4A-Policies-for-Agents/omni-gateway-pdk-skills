# <Policy Name>

<One paragraph: what this policy does and why it exists. No bullet lists, no tables — a single
descriptive paragraph that someone can read to understand the policy's purpose at a glance.>

<!-- Optional, only if your project classifies policies. Drop this line if it doesn't.
**Category**: <your project's taxonomy value — e.g. by protocol or concern>
-->

## Configuration

<One `### <propertyName>` subsection per top-level property in the policy's definition. Use the
property name verbatim so it can be checked against the config schema. Omit this whole section if
the policy has no configuration.>

### <propertyName>

- **Type**: `<string | number | boolean | object | array<...>>`
- **Required**: <yes | no>
- **Default**: `<value, or "none">`

<What it does, in a sentence or two. A concrete example value. Any gotchas inline — don't defer
them to a separate Troubleshooting section.>

## Behavior

<The observable contract. Usually the largest section.>

### Request flow

<What the policy inspects, what it mutates, and the conditions under which it breaks the filter
chain (`Flow::Break`) vs. lets it continue (`Flow::Continue`).>

### Response flow

<Only if the policy touches responses. Delete this subsection otherwise.>

### Rules

<Number the checks the policy enforces, so tests and error messages can reference them by number.>

1. <Rule one.>
2. <Rule two.>

### Missing context (local-mode)

<What happens when the gateway runs without a control-plane connection. Default per the runtime
model: log a warning and continue. Document any stricter, justified override here. Do NOT discuss
the authentication object here — that's a filter-chain ordering concern; mention it under Edge
cases if relevant.>

### Edge cases

- <Bullet per edge case — empty body, oversized payload, malformed input, etc.>

### Error responses

| Condition | HTTP status | Protocol error (if any) |
|---|---|---|
| <when this happens> | `<4xx/5xx>` | `<e.g. JSON-RPC -32602, or n/a>` |

### Benchmarks

<Name the latency budget and the scenarios a Criterion bench covers. Reference the bench file, e.g.
`benches/<name>.rs`. Skip only for a thin pass-through whose cost is harness-dominated.>

## Architecture

<Internals only — components, data flow, extension points, non-obvious dependencies. Include ONLY
when someone changing the code would benefit from a component map. Delete this section for
single-filter policies.>

## Examples

<Complete, runnable config + request/response samples for the common cases. At minimum one
happy-path example. Use correct language tags on fenced blocks.>

```yaml
# policy configuration
```

```http
# example request
```

```http
# example response
```
