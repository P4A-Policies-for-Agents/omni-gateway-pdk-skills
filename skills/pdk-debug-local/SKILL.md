---
name: pdk-debug-local
description: Use when debugging and testing custom policies locally using the PDK Debugging Playground with Omni Gateway in Local Mode in Docker, including registration setup, api.yaml configuration, make run deployment, and testing with curl requests.
---

# Skill: Debugging Custom Policies with the PDK Debugging Playground

## Overview

This skill covers how to debug and test custom policies locally using the PDK Debugging Playground. The playground provides a pre-configured API instance with Omni Gateway running in Local Mode in a Docker container.

**Important:** The PDK Debugging Playground only supports Omni Gateway running in Local Mode in a Docker container. Even if deploying to Connected Mode or a different platform, always test locally first.

## Prerequisites

- **Docker** must be installed and running
- Policy project must be compiled (`make build`)
- A registered Omni Gateway instance in Local Mode

## Steps

### 1. Register an Omni Gateway Instance in Local Mode

A `registration.yaml` file must exist in the `<root-directory>/playground/config` directory.

Create this file by running the Omni Gateway registration command **from inside that directory**, or copy a `registration.yaml` from a previously registered Omni Gateway.

**Important:**
- Run only the registration command — do **not** run the Docker start command (the `make run` command handles that)
- Create a different registration file on each device testing the policy
- The registration file is `.gitignore`d

Registration methods:
- Register and Run with a Username and Password in a Docker Container
- Register and Run with a Connected App in a Docker Container
- Register and Run with a Token in a Docker Container

### 2. Configure the Policy

Based on the properties defined in `definition/gcl.yaml`, configure the required parameters in `/playground/config/api.yaml`.

**Example `gcl.yaml`:**

```yaml
apiVersion: gateway.mulesoft.com.v1alpha1
kind: Extension
metadata:
  labels:
    title: my-custom-policy
    category: Security
spec:
  extends:
    - name: extension-definition
  properties:
    user:
      type: string
      default: user1
    password:
      type: string
    description:
      type: string
  required:
    - user
    - password
```

**Corresponding `api.yaml` configuration:**

```yaml
apiVersion: gateway.mulesoft.com/v1alpha1
kind: ApiInstance
metadata:
  name: ingress-http
spec:
  address: http://0.0.0.0:8081
  services:
    upstream:
      address: http://backend
      routes:
        - config:
            destinationPath: /anything/echo/
  policies:
    - policyRef:
        name: my-custom-policy
      config:
        password: 12345678
```

In this example, `user` defaults to `user1`, `password` is set to `12345678`, and `description` is omitted (not required).

### 3. Add Additional Policies (Optional)

You can apply additional policies to the test API instance to test your custom policy alongside other policies.

**Important:** Your custom policy under development must be the **first** policy in the list:

```yaml
  policies:
    - policyRef:
        name: my-custom-policy
      config:
        password: 12345678
    - policyRef:
        name: additional-policy
      config:
```

### 4. Deploy the Policy with PDK

Run the following command from the policy's root directory:

```bash
make run
```

This starts two Docker containers defined in `<root-directory>/playground/docker-compose.yaml`:

| Container | Purpose |
|---|---|
| `Local-flex` | Omni Gateway instance that executes the custom policy. Listens on `localhost:8081` |
| `Backend` | Backend service that echoes any request it receives |

To stop the containers, press `Cmd+c` or `Ctrl+c` from the terminal running them.

### 5. Test with Requests

Once containers are running, send requests to the API instance:

```bash
curl --location --request POST 'http://0.0.0.0:8081/some/route' \
--header 'Token: mytoken'
```

The backend service returns an echo response including the request's route, body, and headers — with any modifications performed by the custom policy or additional policies.

Policy logs from the source code are visible in the terminal running the containers.

## Configure the Loglevel

By default, the debugging environment enables logs with the `debug` loglevel.

To change the loglevel, edit the `logging.runtimeLogs.logLevel` value in:

```
<root-directory>/playground/config/logging.yaml
```

## Using the Rust Debugger

VS Code includes a Rust debugger that can run unit tests. You **cannot** use the VS Code Debugger while the policy is deployed on Omni Gateway — you must use unit tests for specific logic you need to debug.

To debug Rust code in VS Code:
1. Install the [CodeLLDB Debugger Extension](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) (or equivalent)
2. Set breakpoints in unit test code
3. Use the VS Code debug panel to run tests with the debugger attached

## See Also

- Skill: Writing Integration Tests (`pdk-integration-tests`)
- Skill: Test PDK Policy Locally (`pdk-test-locally`)
