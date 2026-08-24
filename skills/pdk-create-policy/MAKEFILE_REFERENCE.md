# Makefile & Scripts Reference

Reference files for the split-model policy project structure. Use a **freshly generated PDK 1.10
project** as the canonical source. Copying an older Makefile and only changing Cargo versions misses
generated-project behavior added in PDK 1.10.

## Reference Policy

Generate a temporary reference project with the current `anypoint-pdk-plugin`, then copy or compare
the relevant files. Preserve project-specific business-group helpers after the generated baseline is
in place.

```bash
# Compare the generated reference with the target before copying:
diff -u <generated-1.10-project>/Makefile <policy-name>-flex/Makefile
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

PDK 1.10-generated implementation projects also provide:

- `make test TEST=<test_name>` to run one integration test
- Automatic disconnected registration for `playground/config/registration.yaml` and
  `tests/config/registration.yaml` when absent
- A playground image parameterized with `PDK_TEST_FLEX_IMAGE_NAME` and
  `PDK_TEST_FLEX_IMAGE_VERSION`
These are generated-file features. Updating `pdk`, `pdk-test`, or `cargo-anypoint` does not inject
them into an existing Makefile or Compose file.

For **unified-model projects generated with PDK 1.10 or later** only, publish and release reuse an
unchanged policy definition by default. Set `SKIP_UNCHANGED_DEFINITION=false` to publish a new
definition. Do not backport this flag to an earlier-generated project; the feature is unsupported
for projects created before PDK 1.10. Split-model implementation Makefiles publish the policy WASM
and do not own this unified definition-reuse setting.

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
