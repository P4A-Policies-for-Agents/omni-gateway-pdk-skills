# omni-gateway-pdk-skills

A curated collection of [Claude Code](https://claude.com/claude-code) **skills** for developing
custom **MuleSoft Flex Gateway** policies with the **Policy Development Kit (PDK)**.

Each skill is an installable guide that coaches an AI coding agent (or a human reading along)
through one slice of the PDK policy lifecycle — scaffolding a project, writing idiomatic Rust,
implementing a specific policy feature, testing locally, and upgrading the PDK. The content is
derived from the public [PDK documentation](https://docs.mulesoft.com/pdk/latest/) and distilled
into focused, task-oriented playbooks.

## What's inside

Skills live under [`skills/`](skills/), one directory per skill (`pdk-<topic>/SKILL.md`). They
fall into a few groups:

- **Getting started** — `pdk-create-policy`, `pdk-code-style`, `pdk-coding-best-practices`
- **Testing & debugging** — `pdk-unit-tests`, `pdk-integration-tests`, `pdk-test-locally`,
  `pdk-debug-local`, `pdk-troubleshooting`
- **Policy features** — authentication, JWT, CORS, rate limiting, caching, IP filtering, schema
  and contract validation, DataWeave, HTTP calls, WebSockets, and more
- **AI gateway** — `pdk-mcp`, `pdk-a2a`, `pdk-embedding-services`, `pdk-vector-stores`
- **Maintenance** — `pdk-upgrade-pdk`

Many `SKILL.md` files end with a **Source Ref** block recording the public docs page and snapshot
date the content was derived from.

## Using these skills

Clone or copy the `skills/` directory into a location your agent loads skills from (for Claude
Code, a plugin or your skills directory). Once installed, the agent invokes the relevant
`pdk-*` skill automatically when you work on a PDK policy.

## Versioning

Skills currently target **PDK 1.9.0**. PDK ships breaking changes between releases (CLI plugin
renames, Rust/WASI target bumps); see `skills/pdk-upgrade-pdk/SKILL.md` for upgrade guidance.

## License

[Apache License 2.0](LICENSE). Copyright (c) 2026 Salesforce, Inc.
