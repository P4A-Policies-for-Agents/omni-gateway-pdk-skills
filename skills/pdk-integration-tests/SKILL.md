---
name: pdk-integration-tests
description: Use when writing integration tests for PDK custom policies using Docker-based Omni Gateway in Local Mode, configuring FlexConfig and ApiConfig, mocking backend services with HttpMockConfig, or troubleshooting policy loading issues in the automated testing framework.
---

# Skill: Writing Integration Tests

## Topic: Testing

This skill covers how to write integration tests for PDK custom policies using the automated testing framework with Docker-based Omni Gateway in Local Mode.

## Prerequisites

- Docker must be installed
- Omni Gateway must be registered in Local Mode (registration.yaml in `tests/config/`)

## Integration Tests Directory Structure

```
tests
├── requests.rs
└── common
    └── mod.rs
└── config
    └── logging.yaml
    └── registration.yaml
```

- `requests.rs`: Individual test module and executable
- `common/mod.rs`: Shared functionalities, API configurations, and policy configurations included in every test module

## Register Omni Gateway for Testing

A `registration.yaml` file must exist in `<root-directory>/tests/config`. Create it by running the Omni Gateway registration command in that directory, or move an existing one there.

**Important:** Run only the registration command — do not run the Docker start command (`make test` handles that).

The registration file is `.gitignore`d. Create a different one on each device.

## Configure the Policy Under Test

1. Get the policy ID:
   ```sh
   make show-policy-ref-name
   ```

2. Set it in `common/mod.rs`:
   ```rust
   pub const POLICY_NAME: &str = "<custom-policy-id>";
   ```

3. Configure policy parameters (see `gcl.yaml` for available parameters).

## Create a Test Function

Test functions are `async` functions decorated with `#[pdk_test]`:

```rust
use pdk_test::pdk_test;

#[pdk_test]
async fn say_hello() {
    // empty test — always passes
}
```

## Run Integration Tests

```sh
make test
```

The `make test` command compiles the policy before running the tests.

## Configure an httpmock Service

Use `HttpMockConfig` to configure a mock backend server:

```rust
use pdk_test::{pdk_test, TestComposite};
use pdk_test::services::httpmock::HttpMockConfig;

#[pdk_test]
async fn say_hello() -> anyhow::Result<()> {
    let backend_config = HttpMockConfig::builder()
        .hostname("backend")
        .port(80)
        .build();

    let composite = TestComposite::builder()
        .with_service(backend_config)
        .build()
        .await?;

    Ok(())
}
```

To use a custom Docker image: `.image_name("myrepo/httpmock")`.

## Get a Service Handle

Every configured service has a handle for custom interaction:

```rust
use pdk_test::services::httpmock::{HttpMock, HttpMockConfig};

let httpmock: HttpMock = composite.service()?;
```

## Mock Endpoints and Make Requests

Use the [httpmock When/Then API](https://docs.rs/httpmock/latest/httpmock/#whenthen-api) with `reqwest` for HTTP requests:

```rust
use pdk_test::{pdk_test, TestComposite};
use pdk_test::services::httpmock::{HttpMock, HttpMockConfig};

#[pdk_test]
async fn say_hello() -> anyhow::Result<()> {
    let backend_config = HttpMockConfig::builder()
        .hostname("backend")
        .port(80)
        .build();

    let composite = TestComposite::builder()
        .with_service(backend_config)
        .build()
        .await?;

    let httpmock: HttpMock = composite.service()?;
    let mock_server = httpmock::MockServer::connect_async(httpmock.socket()).await;

    mock_server.mock_async(|when, then| {
        when.path_contains("/hello");
        then.status(202).body("World!");
    }).await;

    let base_url = mock_server.base_url();
    let response = reqwest::get(format!("{base_url}/hello")).await?;

    assert_eq!(response.status(), 202);
    assert_eq!(response.text().await?, "World!");

    Ok(())
}
```

## Configure a Flex Service Instance

`FlexConfig` properties:

| Property | Description |
|---|---|
| `version` | Omni Gateway version to test |
| `image_name` | Docker image name (default: `mulesoft/flex-gateway`) |
| `hostname` | Hostname (default: `local-flex`) |
| `config_mounts` | Map of local directories for configuration files |
| `with_api` | API configuration deployed to the Flex instance (call multiple times for multiple APIs) |
| `ports` | Ports where the Flex service listens |

`ApiConfig` properties:

| Property | Description |
|---|---|
| `name` | Name of the API |
| `upstream` | Reference to the upstream service |
| `path` | Path for upstream requests |
| `port` | Port where the API listens |
| `policies` | List of `PolicyConfig` instances |

`PolicyConfig` properties:

| Property | Description |
|---|---|
| `name` | Policy to apply |
| `configuration` | Parameters for the policy |

```rust
mod common;
use common::*;

use pdk_test::{pdk_test, TestComposite};
use pdk_test::port::Port;
use pdk_test::services::flex::FlexConfig;

const FLEX_PORT: Port = 8081;

#[pdk_test]
async fn say_hello() -> anyhow::Result<()> {
    let httpmock_config = HttpMockConfig::builder()
        .port(80)
        .version("latest")
        .hostname("backend")
        .build();

    let policy_config = PolicyConfig::builder()
        .name(POLICY_NAME)
        .configuration(serde_json::json!({"source": "http://backend/blocked", "frequency": 60}))
        .build();

    let api_config = ApiConfig::builder()
        .name("ingress-http")
        .upstream(&httpmock_config)
        .path("/anything/echo/")
        .port(FLEX_PORT)
        .policies([policy_config])
        .build();

    let flex_config = FlexConfig::builder()
        .version("1.6.1")
        .hostname("local-flex")
        .config_mounts([
            (POLICY_DIR, "policy"),
            (COMMON_CONFIG_DIR, "common")
        ])
        .with_api(api_config)
        .build();

    let composite = TestComposite::builder()
        .with_service(flex_config)
        .with_service(httpmock_config)
        .build()
        .await?;

    Ok(())
}
```

To use a custom Flex Docker image: `.image_name("myrepo/flex-gateway")`.

## Flex Service Endpoint Requests

Access Flex endpoints via external URLs indexed by port:

```rust
use pdk_test::services::flex::Flex;

let flex: Flex = composite.service()?;
let flex_url = flex.external_url(FLEX_PORT).unwrap();
let response = reqwest::get(format!("{flex_url}/hello")).await?;
```

## Complete Example: Flex + httpmock

```rust
mod common;

use httpmock::MockServer;
use pdk_test::{pdk_test, TestComposite};
use pdk_test::port::Port;
use pdk_test::services::flex::{FlexConfig, Flex};
use pdk_test::services::httpmock::{HttpMockConfig, HttpMock};

use common::*;

const HELLO_CONFIG_DIR: &str = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/requests/hello");
const FLEX_PORT: Port = 8081;

#[pdk_test]
async fn hello() -> anyhow::Result<()> {
    // Configure HttpMock backend
    let httpmock_config = HttpMockConfig::builder()
        .port(80)
        .version("latest")
        .hostname("backend")
        .build();

    // Configure policy
    let policy_config = PolicyConfig::builder()
        .name(POLICY_NAME)
        .configuration(serde_json::json!({"source": "http://backend/blocked", "frequency": 60}))
        .build();

    // Configure API
    let api_config = ApiConfig::builder()
        .name("ingress-http")
        .upstream(&httpmock_config)
        .path("/anything/echo/")
        .port(FLEX_PORT)
        .policies([policy_config])
        .build();

    // Configure Flex
    let flex_config = FlexConfig::builder()
        .version("1.6.1")
        .hostname("local-flex")
        .config_mounts([
            (POLICY_DIR, "policy"),
            (COMMON_CONFIG_DIR, "common")
        ])
        .with_api(api_config)
        .build();

    // Compose services
    let composite = TestComposite::builder()
        .with_service(flex_config)
        .with_service(httpmock_config)
        .build()
        .await?;

    let flex: Flex = composite.service()?;
    let flex_url = flex.external_url(FLEX_PORT).unwrap();

    let httpmock: HttpMock = composite.service()?;
    let mock_server = MockServer::connect_async(httpmock.socket()).await;

    mock_server.mock_async(|when, then| {
        when.path_contains("/hello");
        then.status(202).body("World!");
    }).await;

    let response = reqwest::get(format!("{flex_url}/hello")).await?;
    assert_eq!(response.status(), 202);

    Ok(())
}
```

## Reviewing Service Logs

After a test failure, check service logs at:
```
<root-directory>/target/pdk-test/<module-name>/<test>/<service>.log
```

## Troubleshooting: Policy Not Loading in Tests (UNVERIFIED)

> **Note:** These findings were discovered during debugging of the `google-cloud-auth` policy and may not apply universally. Treat as hypotheses to verify on a case-by-case basis.

If tests return unexpected 200 (pass-through) instead of expected policy behavior, the policy may not be loading. Possible causes:

### 1. Stale Version Artifacts in Target

The `target/wasm32-wasip1/release/<group-id>/` directory may contain artifacts from previous versions. Omni Gateway reads all GCL files from the mounted directory, and old definitions can conflict. Try removing stale version directories:
```bash
rm -rf target/wasm32-wasip1/release/<group-id>/<asset-name>/<old-version>
```

### 2. POLICY_NAME Version Mismatch

`POLICY_NAME` in `tests/common/mod.rs` must match the version-derived name. Version `1.x.y` generates `<name>-v1-0`, version `2.x.y` generates `<name>-v2-0`. After version changes, check `target/policy-ref-name.txt` and update `POLICY_NAME` accordingly.

### 3. Service URL Path Mismatch in Mocks

When a policy makes HTTP calls to an external service (e.g., token endpoint), the `format: service` property splits URLs into host:port (Service) and path (passed via `.path()`). If the test config URL doesn't include the path, httpmock mocks matching on specific paths won't match:
```rust
// Wrong - no path, mock for /token won't match
"tokenUri": format!("http://{}:{}", config.hostname(), config.port()),

// Correct - includes /token path
"tokenUri": format!("http://{}:{}/token", config.hostname(), config.port()),
```

### 4. Capturing Omni Gateway Logs

To debug policy loading issues, check the Flex container logs:
```bash
docker ps -a --filter "ancestor=mulesoft/flex-gateway:<version>" --format "{{.ID}}" | head -1 | xargs docker logs 2>&1 | grep -i error
```

### 5. Recommended Omni Gateway Version

Use Omni Gateway **1.12.1** or later for integration tests.

## Documentation Reference

- Source: https://docs.mulesoft.com/pdk/latest/policies-pdk-integration-tests

## Source Ref

- **Repo:** `mulesoft/docs-gateway` @ `8cafed6`
- **Branch:** `latest`
- **File:** `pdk/1.8/modules/ROOT/pages/policies-pdk-integration-tests.adoc`
- **Snapshot:** 2026-05-14
