# Makefile & Scripts Reference

Reference files for the split-model policy project structure. When creating a new policy, **copy these from an existing policy** (e.g., `slack-request-verification`) rather than recreating them.

## Reference Policy

Use `policies/slack-request-verification/` as the canonical source:

```bash
# From the new policy's parent directory:
cp -r ../slack-request-verification/scripts ./scripts
cp ../slack-request-verification/slack-request-verification-definition/Makefile <policy-name>-definition/Makefile
cp ../slack-request-verification/slack-request-verification-flex/Makefile <policy-name>-flex/Makefile
```

## What These Files Provide

### `scripts/select-bg.sh`

Interactive business group selector shared by both definition and implementation Makefiles.

- Fetches available BGs via `anypoint-cli-v4 account business-group list`
- Displays a numbered menu with `▶` marking the currently configured BG
- Updates the `groupId` in `exchange.json` (definition) or `group_id` in `Cargo.toml` (implementation)
- Switches the CLI context via `anypoint-cli-v4 conf organization <bg-id>` (required for child BG publishing)
- Flushes stdin before prompting to avoid stale input from piped Make commands

### Definition Makefile (`<policy-name>-definition/Makefile`)

Targets:
- `build` — Build the policy definition
- `publish` — Publish definition to Exchange as `-DEV` asset
- `release` — Publish definition to Exchange (production)
- `release-local` — Publish definition to local filesystem
- `release-interactive` — Select BG interactively, then release

### Implementation Makefile (`<policy-name>-flex/Makefile`)

Targets:
- `setup` — Install cargo-anypoint
- `build-asset-files` — Fetch definition from Exchange and generate config
- `build` — Build the WASM policy implementation
- `run` — Run policy in local Docker playground
- `test` — Run integration tests
- `publish` — Publish implementation to Exchange as dev version
- `release` — Publish implementation to Exchange (production)
- `release-interactive` — Select BG interactively, then release

## `.gitignore` entries

The implementation `.gitignore` should include:
```
target
playground/config/custom-policies/*
!playground/config/custom-policies/note.txt
.pdk
registration.yaml
certificate.yaml
```
