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
- **File:** `pdk/1.10/modules/ROOT/pages/policies-pdk-publish-policies.adoc`
- **Snapshot:** 2026-08-24
