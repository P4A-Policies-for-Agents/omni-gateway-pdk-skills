---
name: pdk-test-locally
description: Use when testing a PDK custom policy locally using the Docker playground with Omni Gateway, including setup, build, curl testing, iteration, and debugging integration tests with pdk-test framework.
---

# Skill: Test PDK Policy Locally

## Overview

This skill guides local testing of any Omni Gateway custom policy using the Docker playground and curl. It applies to both unified and split-model project structures.

## Prerequisites

- **Docker** must be running
- **Rust toolchain** with `wasm32-wasip1` target installed (via rustup, not Homebrew)
- **cargo-anypoint** installed (`make setup` in the implementation directory)
- **`registration.yaml`** must exist under `playground/config/` (copy from another policy if missing)

## Steps

### 1. Verify Docker is Running

```bash
docker info > /dev/null 2>&1 && echo "Docker is running" || echo "Docker is NOT running — start Docker Desktop first"
```

If Docker is not running, start Docker Desktop and wait for it to be ready before proceeding.

### 2. Locate the Implementation Directory

- **Unified project**: the policy root directory (e.g., `policies/circuit-breaker/`)
- **Split-model project**: the `-flex` subdirectory (e.g., `policies/<policy-name>/<policy-name>-flex/`)

All subsequent commands run from this directory.

### 3. Verify Playground Prerequisites

Ensure the following files exist under `playground/config/`:

- **`api.yaml`** — API instance config with policy reference and test configuration values
- **`registration.yaml`** — Omni Gateway registration (contains agent ID, certificates, platform URLs). Copy from an existing policy if missing:

```bash
cp ../../<existing-policy>/playground/config/registration.yaml playground/config/registration.yaml
# or for split-model:
cp ../../<existing-policy>/<existing-policy>-flex/playground/config/registration.yaml playground/config/registration.yaml
```

### 4. Build and Run

```bash
make run
```

This will:
1. Build the WASM binary (`cargo build --target wasm32-wasip1 --release`)
2. Generate policy definition and implementation GCL files
3. Install the policy into `playground/config/custom-policies/`
4. Patch `api.yaml` with the correct policy reference name
5. Start Docker Compose with Omni Gateway (port 8081) and backend service(s)

Wait for the Omni Gateway logs to show it is ready (look for `Omni Gateway is running`) before sending requests.

### 5. Test with curl

The Omni Gateway listens on `http://localhost:8081`. Send requests to the route configured in `playground/config/api.yaml` (typically `/anything/echo/`).

```bash
# Basic GET request
curl -v http://localhost:8081/anything/echo/test

# POST with body
curl -v -X POST http://localhost:8081/anything/echo/test \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# With custom headers (for policies that inspect headers)
curl -v http://localhost:8081/anything/echo/test \
  -H "X-Custom-Header: some-value"
```

Adjust headers, body, and method based on what the specific policy inspects.

### 6. Verify Policy Behavior

- **Request allowed**: expect 200 (or whatever the backend returns)
- **Request rejected by policy**: expect the policy's error status code (e.g., 401, 403, 429, 503) with a JSON error body
- **Check logs**: Omni Gateway logs stream in the Docker Compose terminal. Look for `logger::info!`, `logger::warn!`, and `logger::debug!` output from the policy

### 7. Iterate

To make changes and re-test:
1. Stop the playground: `Ctrl+C` or `docker compose -f ./playground/docker-compose.yaml down`
2. Edit source code in `src/lib.rs`
3. Run `make run` again (it rebuilds automatically)

### 8. Run Integration Tests

```bash
make test
```

This builds the policy and runs the test suite in `tests/requests.rs` using `pdk-test` with a Docker-based Omni Gateway and mock backend.

### 9. Stop the Playground

Press `Ctrl+C` in the terminal running `make run`, or from another terminal:

```bash
docker compose -f ./playground/docker-compose.yaml down
```

## Common Issues

- **`wasm32-wasip1` target not found**: Homebrew Rust doesn't support WASM targets. Ensure `$HOME/.cargo/bin` is first in PATH (rustup-managed toolchain).
- **`make run` fails immediately**: Check that `registration.yaml` exists in `playground/config/`.
- **Port 8081 already in use**: Stop other services using port 8081, or change the port in both `playground/config/api.yaml` and `playground/docker-compose.yaml`.
- **Policy not applied**: Verify `playground/config/api.yaml` has the correct `policyRef.name` and matching `config` values. The `make run` target auto-patches the policy ref name.
