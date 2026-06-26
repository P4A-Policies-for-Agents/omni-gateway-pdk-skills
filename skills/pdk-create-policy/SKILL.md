---
name: pdk-create-policy
description: Use when scaffolding new PDK custom policies with the split-model structure using anypoint-cli-v4 pdk policy-project create to generate definition projects with gcl.yaml schema and implementation projects with Rust entrypoint, filters, playground Docker setup, and Exchange publishing via Makefiles with business group selection.
---

# Skill: Create PDK Custom Policy (Split-Model)

## Overview

This skill guides the creation of new Omni Gateway custom policies using the PDK **split-model** project structure. The split-model separates the policy **definition** (schema/configuration) and **implementation** (Rust logic) into two distinct projects, each scaffolded with its own CLI command.

## Prerequisites

- **Anypoint CLI v4** (v1.6.14+): `npx anypoint-cli-v4@latest`
- **PDK Plugin** (v1.7.0+): `anypoint-cli-v4 plugins:install anypoint-pdk-plugin`
  - **Important**: If the older `anypoint-cli-pdk-plugin` is installed, uninstall it first (`anypoint-cli-v4 plugins:uninstall anypoint-cli-pdk-plugin`) — it conflicts with the new plugin and intercepts the `--project-mode` flag.
- **Rust toolchain**: edition 2018+, with `wasm32-wasip1` target installed
- **Docker**: required for local playground testing
- **Anypoint Platform account**: with organization access for publishing

## Workflow

### 1. Gather Requirements

Before scaffolding, ask the user for:
- **Policy name** (kebab-case, e.g., `rate-limiter`)
- **Title and description** for the policy definition
- **Category** (e.g., Security, Resilience, Transformation, Quality of Service, Compliance, A2A)
- **Interface scope**: `api`, `resource`, or `api,resource`
- **Configuration properties**: name, type, default value, description, constraints
- **Behavior**: what the policy does on request and/or response (filter, transform, reject, log, etc.)

### 2. Create Parent Directory

Create a parent folder under `policies/` named after the policy:

```bash
mkdir policies/<policy-name>
cd policies/<policy-name>
```

### 3. Scaffold Definition Project

```bash
anypoint-cli-v4 pdk policy-project create --name <policy-name> --project-mode definition \
  --category <Category> --interface-scope <scope> --description "<description>"
```

The `--category`, `--interface-scope`, and `--description` flags are optional but recommended to pre-populate the gcl.yaml.

This creates `<policy-name>-definition/` containing:
- `gcl.yaml` — Policy definition schema
- `exchange.json` — Exchange asset coordinates (groupId, assetId, version, name)
- `Makefile` — build, publish, release, release-local, release-interactive
- `AGENTS.md` — guidance for AI coding agents (repo purpose, structure, proxy-wasm constraints, coding rules, pitfalls, doc links). Scaffolded by PDK 1.9.0+.
- `.gitignore`

### 4. Define Policy Schema

Edit `<policy-name>-definition/gcl.yaml` following this structure:

```yaml
---
apiVersion: gateway.mulesoft.com/v1alpha1
kind: Extension
metadata:
  labels:
    title: <Policy Title>
    description: <Policy description>
    category: <Category>
    metadata/interfaceScope: api
spec:
  extends:
    - name: extension-definition
      namespace: default
  properties:
    <propertyName>:
      type: string          # string, integer, boolean, array, object
      default: "value"
      description: Description of the property
    <anotherProperty>:
      type: integer
      default: 10
      description: Another property
      minimum: 1
      maximum: 100
  required:
    - <propertyName>
```

**Supported property types:** `string`, `integer`, `boolean`, `array`, `object`
**Property modifiers:** `default`, `description`, `enum`, `minimum`, `maximum`, `items` (for arrays), `properties` (for objects), `required` (for objects), `uniqueItems`

### 5. Create Shared Scripts Directory

Create a `scripts/` directory in the policy parent folder with the `select-bg.sh` helper script for interactive BG selection:

```bash
mkdir -p scripts
```

Create `scripts/select-bg.sh` with the following content:

```bash
#!/usr/bin/env bash
# Interactive BG selector for policy release targets.
# Usage: select-bg.sh <config-file> (exchange.json or Cargo.toml)

set -euo pipefail

CONFIG_FILE="${1:?Usage: select-bg.sh <config-file>}"

echo "Fetching business groups..."
BG_JSON=$(anypoint-cli-v4 account business-group list -o json)

BG_NAMES=()
BG_IDS=()
while IFS= read -r line; do BG_NAMES+=("$line"); done < <(echo "$BG_JSON" | python3 -c "import sys,json; [print(bg['name']) for bg in json.load(sys.stdin)]")
while IFS= read -r line; do BG_IDS+=("$line"); done < <(echo "$BG_JSON" | python3 -c "import sys,json; [print(bg['id']) for bg in json.load(sys.stdin)]")

# Read current groupId
if [[ "$CONFIG_FILE" == *.json ]]; then
  CURRENT_BG=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE'))['groupId'])")
elif [[ "$CONFIG_FILE" == *.toml ]]; then
  CURRENT_BG=$(python3 -c "import tomllib; print(tomllib.load(open('$CONFIG_FILE','rb'))['package']['metadata']['anypoint']['group_id'])")
else
  echo "Unsupported config file: $CONFIG_FILE" >&2
  exit 1
fi

echo ""
echo "Available Business Groups:"
echo "─────────────────────────────────────────────────"
for i in "${!BG_NAMES[@]}"; do
  MARKER="  "
  if [ "${BG_IDS[$i]}" = "$CURRENT_BG" ]; then MARKER="▶ "; fi
  echo "  ${MARKER}$((i+1))) ${BG_NAMES[$i]} (${BG_IDS[$i]})"
done
echo "─────────────────────────────────────────────────"
echo ""

printf "Select a BG [1-${#BG_NAMES[@]}]: "
while read -r -t 0; do read -r -n 256; done
read -r choice

IDX=$((choice - 1))
if [ "$IDX" -lt 0 ] || [ "$IDX" -ge "${#BG_NAMES[@]}" ]; then
  echo "Invalid selection."
  exit 1
fi

SELECTED_ID="${BG_IDS[$IDX]}"
SELECTED_NAME="${BG_NAMES[$IDX]}"

echo ""
echo "Selected: $SELECTED_NAME ($SELECTED_ID)"

# Update config file
if [[ "$CONFIG_FILE" == *.json ]]; then
  python3 -c "import json; d=json.load(open('$CONFIG_FILE')); d['groupId']='$SELECTED_ID'; json.dump(d, open('$CONFIG_FILE','w'), indent=2)"
elif [[ "$CONFIG_FILE" == *.toml ]]; then
  sed -i '' "s/^group_id = \".*\"/group_id = \"$SELECTED_ID\"/" "$CONFIG_FILE"
fi

anypoint-cli-v4 conf organization "$SELECTED_ID"
echo "Switched CLI context to: $SELECTED_NAME ($SELECTED_ID)"
echo "Updated $CONFIG_FILE with groupId: $SELECTED_ID"
echo ""
```

Make the script executable:
```bash
chmod +x scripts/select-bg.sh
```

This script:
- Fetches available BGs dynamically via `anypoint-cli-v4 account business-group list`
- Displays a numbered menu with `▶` marking the currently configured BG
- Updates the `groupId` in `exchange.json` (definition) or `group_id` in `Cargo.toml` (implementation)
- Switches the CLI context to the selected BG via `anypoint-cli-v4 conf organization` (required for child BG publishing)

### 6. Build & Optionally Publish Definition

```bash
cd <policy-name>-definition
make build
```

For local development (no Exchange publish), use:
```bash
make release-local
```

For publishing to Exchange:
```bash
make publish              # development version
make release              # production release (current BG)
make release-interactive  # select BG interactively, then release
```

### 7. Scaffold Implementation Project

```bash
cd policies/<policy-name>
anypoint-cli-v4 pdk policy-project create --name <policy-name> --project-mode implementation
```

To scaffold a **FIPS 140-3 compliant** implementation project, add the `--fips` flag (PDK 1.9.0+):

```bash
anypoint-cli-v4 pdk policy-project create --name <policy-name> --project-mode implementation --fips
```

This wires the project to FIPS-compliant cryptographic modules. Only use it when the policy must meet FIPS 140-3 requirements.

This creates `<policy-name>-flex/` (not `-implementation`) containing:
- `Cargo.toml` — Rust manifest with `[package.metadata.anypoint]`
- `src/lib.rs` — Policy implementation stub
- `src/generated/config.rs` and `mod.rs` — Auto-generated from gcl.yaml
- `playground/` — Docker-based local testing environment
- `tests/requests.rs` — Integration test scaffold
- `Makefile` — setup, build, run, test, publish, release, release-interactive
- `AGENTS.md` — guidance for AI coding agents (proxy-wasm runtime constraints, coding rules, common pitfalls, doc links). Scaffolded by PDK 1.9.0+.

### 8. Setup Implementation

```bash
cd <policy-name>-flex
make setup
```

### 9. Link Definition to Implementation

Edit `Cargo.toml` and ensure `definition_asset_id.name` matches the definition's `exchange.json` `assetId`:

```toml
[package.metadata.anypoint]
group_id = "<your-org-group-id>"
definition_asset_id = { name = "<policy-name>", version = "1.0.0" }
implementation_asset_id = "<policy-name>-flex"
```

If working with a development version of the definition, append `-DEV` suffix:
```toml
definition_asset_id = { name = "<policy-name>", version = "1.0.0-DEV" }
```

### 10. Generate Config from Definition

```bash
make build-asset-files
```

This generates/updates `src/generated/config.rs` with a `Config` struct matching the `gcl.yaml` properties. Re-run this command whenever the definition changes.

### 11. Implement Policy Logic

Edit `src/lib.rs`. The implementation follows this pattern:

```rust
mod generated;

use crate::generated::config::Config;
use anyhow::{anyhow, Result};
use pdk::hl::*;
use pdk::logger;

#[entrypoint]
async fn configure(
    launcher: Launcher,
    Configuration(bytes): Configuration,
) -> Result<()> {
    let config: Config = serde_json::from_slice(&bytes).map_err(|err| {
        anyhow!(
            "Failed to parse configuration '{}'. Cause: {}",
            String::from_utf8_lossy(&bytes),
            err
        )
    })?;

    let filter = on_request(|request_state| {
        request_filter(request_state, &config)
    });

    let filter = filter.on_response(|response_state, request_data| {
        response_filter(response_state, request_data)
    });

    launcher.launch(filter).await?;
    Ok(())
}

async fn request_filter(
    request_state: RequestState,
    config: &Config,
) -> Flow<()> {
    let header_state = request_state.into_headers_state().await;
    // Inspect headers, read body, apply policy logic

    // To allow the request through:
    Flow::Continue(())

    // To reject the request:
    // Flow::Break(Response::new(403).with_body("Forbidden"))
}

async fn response_filter(
    response_state: ResponseState,
    request_data: RequestData<()>,
) {
    // Inspect/modify response if needed
}
```

**Key PDK types and functions:**
- `RequestState` → `.into_headers_state()` → `.into_body_state()` for reading request
- `ResponseState` → `.into_headers_state()` → `.into_body_state()` for reading response
- `Flow::Continue(data)` — pass request/response through
- `Flow::Break(Response)` — short-circuit with a custom response
- `Response::new(status_code).with_headers(vec).with_body(string)` — build responses
- `logger::info!()`, `logger::warn!()`, `logger::debug!()` — logging macros

### 12. Configure Playground

Edit `playground/config/api.yaml` to set test configuration values for local testing:

```yaml
---
apiVersion: gateway.mulesoft.com/v1alpha1
kind: ApiInstance
metadata:
  name: ingress-http
spec:
  address: http://0.0.0.0:8081
  services:
    upstream:
      address: "http://backend"
      routes:
        - config:
            destinationPath: /
  policies:
    - policyRef:
        name: <policy-ref-name>
        namespace: default
      config:
        <propertyName>: <testValue>
```

**Add `registration.yaml`**: Copy `playground/config/registration.yaml` from an existing policy in the repo. This file contains the Omni Gateway registration configuration (agent ID, certificates, platform connection URLs) required for the local Docker playground to run. Without it, `make run` will fail.

```bash
cp ../../../<existing-policy>/<existing-policy>-flex/playground/config/registration.yaml \
   playground/config/registration.yaml
```

### 13. Update `.gitignore`

Ensure the implementation `.gitignore` includes entries to exclude build artifacts and sensitive playground files:

```gitignore
target
playground/config/custom-policies/*
!playground/config/custom-policies/note.txt
.pdk
registration.yaml
certificate.yaml
```

**Important**: `registration.yaml` and `certificate.yaml` contain secrets (TLS certificates and keys) and must NOT be committed to git.

### 14. Test

```bash
# Run locally with Docker playground
make run

# Run integration tests
make test
```

### 15. Publish

```bash
make publish              # development version to Exchange
make release              # production release to Exchange (current BG)
make release-interactive  # select BG interactively, then release
```

## Split-Model Project Structure

```
<policy-name>/
  <policy-name>-definition/
    gcl.yaml           # Policy definition schema (GCL)
    exchange.json      # Exchange asset coordinates
    Makefile           # build, publish, release, release-local, release-interactive
    AGENTS.md          # AI coding agent guidance (PDK 1.9.0+)
    .gitignore
  scripts/
    select-bg.sh       # Interactive BG selector (shared by both Makefiles)
  <policy-name>-flex/
    Cargo.toml         # Rust manifest with [package.metadata.anypoint]
    src/
      lib.rs           # Policy implementation (entrypoint + filters)
      generated/
        config.rs      # Auto-generated config struct from gcl.yaml
        mod.rs
    playground/
      config/
        api.yaml           # Local test API config with policy settings
        registration.yaml  # Omni Gateway registration (copied from existing policy, gitignored)
      docker-compose.yaml
    tests/
      common/mod.rs    # Shared test constants (POLICY_DIR, POLICY_NAME, etc.)
      requests.rs      # Integration tests
    Makefile           # setup, build, build-asset-files, run, test, publish, release, release-interactive
    AGENTS.md          # AI coding agent guidance (PDK 1.9.0+)
    .gitignore
```

## Cargo.toml Reference

```toml
[package]
name = "<policy_name_underscored>"
version = "1.0.0"
edition = "2018"

[package.metadata.anypoint]
group_id = "<org-group-id>"
definition_asset_id = { name = "<policy-name>", version = "1.0.0" }
implementation_asset_id = "<policy-name>-flex"

[dependencies]
pdk = { version = "1.9.0" }
serde = { version = "1.0", features = ["derive"] }
serde_json = { version = "1.0", default-features = false, features = ["alloc"] }
anyhow = "1.0"

[dev-dependencies]
pdk-test = { version = "1.9.0" }
httpmock = "0.6"
reqwest = "0.11"

[lib]
crate-type = ["cdylib"]

[profile.release]
lto = true
opt-level = 'z'
strip = "debuginfo"
```

## Gotchas & Lessons Learned

- **Implementation directory naming**: The CLI names the implementation directory `<policy-name>-flex`, not `<policy-name>-implementation`.
- **Plugin conflict**: The older `anypoint-cli-pdk-plugin` conflicts with the newer `anypoint-pdk-plugin`. Uninstall the old one before using `--project-mode`.
- **Child BG publishing**: The CLI defaults to the root org context. To publish to a child BG, switch the CLI context first with `anypoint-cli-v4 conf organization <bg-id>`. The `select-bg.sh` script handles this automatically. Do NOT use the `--organization` flag — it has a CLI bug (TypeError).
- **No `pdk::time` module**: PDK does not expose a time module. Use `std::time::SystemTime` for timestamps (it works in the WASM environment).
- **Response body as bytes**: `Response::new(status).with_body()` expects `Vec<u8>`, not `&str`. Use `.into_bytes()` on strings.
- **Header names are lowercase**: When reading headers via `handler.header("name")`, always use lowercase header names.

## Documentation References

- Split-model project: https://docs.mulesoft.com/pdk/latest/policies-pdk-create-project-split
- Policy templates/examples: https://docs.mulesoft.com/pdk/latest/policies-pdk-policy-templates
- Developing custom policies: https://docs.mulesoft.com/pdk/latest/policies-pdk-develop-custom-policies
- PDK crate: https://crates.io/crates/pdk
- PDK API docs: https://docs.rs/pdk/1.9.0/pdk/
