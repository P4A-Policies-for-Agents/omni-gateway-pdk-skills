---
name: pdk-publish-policies
description: Use when publishing or releasing a PDK unified-model custom policy to Exchange, including DEV versus stable assets, versioning, Exchange permissions, child business groups, and PDK 1.10 unchanged-definition reuse with SKIP_UNCHANGED_DEFINITION.
---

# Skill: Publishing Unified-Model Policies to Exchange

## Publish Versus Release

Use `make publish` while a policy version is still under development:

```bash
make publish
```

The DEV asset uses an asset ID ending in `-dev` and a timestamped version. Re-running the command
publishes another development version while preserving earlier versions.

Use `make release` for the definitive stable version:

```bash
make release
```

The released asset has no `-dev` suffix and uses the version from `Cargo.toml` without a timestamp.
That version is immutable in Exchange; bump the policy version before releasing again.

Every publish or release creates a new `stable`-state asset in Exchange
(https://docs.mulesoft.com/exchange/lifecycle#stablet-state).

## Local Mode

Policies that run on Omni Gateway in Local Mode don't need to be published to Exchange. Exchange is still useful for version control, but distributing the policy to Local Mode gateways is your responsibility.

## Split-Model Publish Choreography

For split-model projects (definition and implementation versioned separately; see [[pdk-create-policy]]), the docs sequence is below. The implementation project's `Cargo.toml` `definition_asset_id` version field must match the definition version, with a `-DEV` suffix while testing against a development definition.

**New policy**
1. Definition project: edit `gcl.yaml`, run `make publish`.
2. Implementation project: set `definition_asset_id` version to `1.0.0-DEV`, run `make build-asset-files`, code and test locally, `make publish` for lower-environment testing.
3. After validation: `make release` in the definition project; set `definition_asset_id` to `1.0.0` in the implementation `Cargo.toml`; `make release` in the implementation project.

**Patch update (implementation only)**
1. Bump the patch version in the implementation `Cargo.toml` (for example `1.0.1`) and set `definition_asset_id` to `1.0.0-DEV`; `make build-asset-files`, test, `make publish`.
2. After validation, remove the `-DEV` suffix from `definition_asset_id` and run `make release` in the implementation project.

**Minor or major update (definition and implementation)**
1. Bump the version in the definition `exchange.json` (for example `1.1.0`), edit `gcl.yaml`, `make publish` the definition.
2. Bump the implementation version, set `definition_asset_id` to `1.1.0-DEV`, `make build-asset-files`, test, `make publish`.
3. After validation, remove `-DEV` from `definition_asset_id` and `make release` the implementation. The docs say "release both assets" but list only the implementation step; run `make release` in the definition project first, as in the new-policy flow.

**Local development without Exchange**: run `make release-local` in the definition project, set `definition_asset_id` to the matching version with `-DEV`, run `make build-asset-files`, and test locally. To distribute as a production asset, remove `-DEV` and run `make build`.

Docs: https://docs.mulesoft.com/pdk/latest/policies-pdk-publish-policies-split

## Prerequisites

- Authenticate the Anypoint CLI with a Connected App that has the `Exchange Contributor` scope.
- Compile the policy before publishing.
- Confirm that the CLI organization context is the intended business group. For child business
  groups, switch context with `anypoint-cli-v4 conf organization <bg-id>` before publishing.

## Reuse an Unchanged Definition (PDK 1.10+)

New unified-model projects generated with PDK 1.10 skip republishing an unchanged policy definition
and reuse the existing definition during `make publish` or `make release`. The command reports:

```text
Definition unchanged from published version x.y.z. Skipping definition publish and reusing it.
```

PDK 1.10-generated projects enable this behavior by default. To always publish a new definition,
change the Makefile default to:

```makefile
SKIP_UNCHANGED_DEFINITION ?= false
```

To force a new definition for one command without changing the Makefile:

```bash
make publish SKIP_UNCHANGED_DEFINITION=false
# or
make release SKIP_UNCHANGED_DEFINITION=false
```

This behavior applies only to projects **generated with PDK 1.10 or later**. It is not supported for
projects generated with an earlier PDK and then upgraded, even if you update Rust dependencies,
`cargo-anypoint`, or copy the generated Makefile flag.

## Troubleshooting

- **409/conflict on release**: The stable version already exists. Bump `version` in `Cargo.toml`.
- **Published to the wrong organization**: Set the CLI business-group context before running Make.
- **Definition is published every time**: Confirm the project was generated with PDK 1.10+ and that
  `SKIP_UNCHANGED_DEFINITION` has not been set to `false` in the Makefile or command environment.
- **Permission denied**: Confirm `Exchange Contributor` scope in the target organization.
- **Schema rejected by Exchange**: Validate field lengths and `assetTypes`; see
  [[pdk-schema-definition]].

## Related Skills

- [[pdk-create-policy]] - generated project and Makefile structure
- [[pdk-upgrade-pdk]] - component and generated-file upgrade matrix
- [[pdk-schema-definition]] - Exchange schema limits and asset metadata

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-publish-policies
- Release notes: https://docs.mulesoft.com/release-notes/pdk/pdk-release-notes (1.10.0)

## Source Ref

- **Repo:** `mulesoft/docs-gateway`
- **Branch:** `latest`
- **File:** `pdk/1.11/modules/ROOT/pages/policies-pdk-publish-policies.adoc`
- **Snapshot:** 2026-11-24
