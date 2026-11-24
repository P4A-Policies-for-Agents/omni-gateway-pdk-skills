---
name: pdk-apply-policies
description: Use when applying, binding, updating, or reordering a compiled PDK custom policy on an API instance in Omni Gateway (Flex Gateway), in Connected Mode through API Manager or in Local Mode through declarative files and the PDK playground, including api.yaml spec.policies, PolicyBinding, policyRef name, and custom-policies files.
---

# Skill: Apply a PDK Custom Policy

## Overview

Applying a custom policy is the same as applying an included policy. Its configuration parameters
come from `definition/gcl.yaml`. Compile the policy first (`make build`).

Getting the **policyRef name** or the **file locations** wrong causes most failures. The binding
then silently does nothing, or the gateway rejects the config. Don't guess either one. Read both
from the build output, as shown below.

## The policyRef name

The policyRef name is the **implementation** ID. It is not the gcl.yaml `metadata.name`, and it is
not the crate name. Get it from the policy's root (implementation) directory:

```bash
make show-policy-ref-name          # prints it
cat target/policy-ref-name.txt     # written by every `make build`
```

It also equals `metadata.name` in the generated `<crate>_implementation.yaml`.

The suffix differs by project, which is why you look it up instead of constructing it:

| Source | Name shape | Makefile variable |
| --- | --- | --- |
| Docs (PolicyBinding example) | `<your_custom_policy>-<version>-impl` | n/a |
| Makefiles with a `-impl` suffix | `$(DEFINITION_NAME)-impl`, e.g. `my-policy-v1-0-impl` | `POLICY_REF_NAME = $(DEFINITION_NAME)$(POLICY_REF_NAME_SUFFIX)` |
| Freshly generated PDK 1.10+ project | e.g. `my-policy-flex-v1-0` (no `-impl`) | `POLICY_REF_NAME = $(shell cargo anypoint get-policy-implementation-name)` |

The version segment (`v1-0`) comes from the Cargo major/minor version. After a version bump, read
the name again.

## Apply in Local Mode (playground)

In the PDK playground, `playground/config/` is mounted as the gateway's `conf.d` directory. The
policy files go in `playground/config/custom-policies/`.

### 1. Copy the generated files

`make build` writes both files to `target/wasm32-wasip1/release/`. Copy them like this:

```bash
cp target/wasm32-wasip1/release/<crate>_implementation.yaml playground/config/custom-policies/
cp target/wasm32-wasip1/release/<crate>_definition.yaml     playground/config/custom-policies/
```

`<crate>` is the Cargo package name with underscores (for example, `header_guard`). You don't copy
the `.wasm` file, because the implementation YAML embeds it. `make run` performs this copy for you
(`rm -f playground/config/custom-policies/*.yaml`, then the two `cp` commands).

### 2. List the policy inline on the ApiInstance (preferred)

`playground/config/api.yaml`:

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
        name: <policy-ref-name>       # from `make show-policy-ref-name`
        namespace: default            # the gcl.yaml metadata namespace
      config:
        headerName: x-api-key         # properties from definition/gcl.yaml
```

`policies` is a list under `spec`, at the same level as `address` and `services`. It does not go
under a route. `make run` overwrites the **first** entry's name, so keep the policy under
development at `spec.policies[0]` and put any additional policies after it. Depending on how the
project was generated, `make run` patches it in one of two ways:

```makefile
# Makefiles that use patch-gcl
anypoint-cli-v4 pdk patch-gcl -f playground/config/api.yaml -p "spec.policies[0].policyRef.name" -v "$(POLICY_REF_NAME)"
anypoint-cli-v4 pdk patch-gcl -f playground/config/api.yaml -p "spec.policies[0].policyRef.namespace" -v "$(DEFINITION_NAMESPACE)"
# Freshly generated PDK 1.10+ Makefile
cargo anypoint patch-api -o playground/config/api.yaml -m $(DEFINITION_GCL_PATH) -n $(POLICY_REF_NAME) -s $(DEFINITION_NAMESPACE)
```

If you start the gateway any way other than `make run`, set the name by hand. Then run `make run`,
or bring the playground up yourself (see `pdk-test-locally`).

### 3. Alternative: a PolicyBinding file (as documented)

This is the docs' Local Mode procedure for a gateway outside the playground:

1. Copy `<your_custom_policy>_definition.yaml` and `<your_custom_policy>_implementation.yaml` from
   `target/wasm32-wasip1/release` to `/etc/mulesoft/flex-gateway/conf.d/`.
2. Create a `.yaml` file with any name in `/etc/mulesoft/flex-gateway/conf.d/custom`. That
   directory can hold multiple configuration files.

```yaml
apiVersion: gateway.mulesoft.com/v1alpha1
kind: PolicyBinding
metadata:
  name: <custom-policy-id>            # the policy instance name
spec:
  targetRef:
    kind: ApiInstance
    name: <your-api-instance>         # must already be defined and applied
  policyRef:
    kind: Extension
    name: <policy-ref-name>           # from `make show-policy-ref-name`
  config:
    headerName: x-api-key
```

The gateway refreshes its configuration automatically when you save the file. Use either the inline
entry or the PolicyBinding for an API, not both.

The paths differ in Docker. The playground Compose file mounts `./config` at
`/usr/local/share/mulesoft/flex-gateway/conf.d/`. On a Linux install the path is
`/etc/mulesoft/flex-gateway/conf.d/`.

### Update in Local Mode

- If the configuration parameters didn't change, replace the old `<your_custom_policy>_implementation.yaml`
  with the new one from `target/wasm32-wasip1/release`. In the playground, run `make run` again.
- If the parameters did change, repeat the full Local Mode apply steps.

## Apply in Connected Mode (API Manager)

1. Publish the policy to Exchange first (see `pdk-publish-policies`). In P4A workspaces, publish
   only through the P4A MCP server.
2. Apply it in API Manager like any included policy, on an API instance of a connected Omni
   Gateway. The `gcl.yaml` parameters and metadata appear as the policy's form in the UI. No local
   files or policyRef name are involved.

To update a custom policy in API Manager:

1. Go to **Anypoint Platform > API Manager > API Administration** and click the API.
2. Click **Policies**. Open the policy's more options menu and click **Edit configuration**.
3. Set **Policy version** to the **latest** version and click **Save**.
4. Open the more options menu again, click **Check for implementation updates**, and then click
   **Update all**.

## Execution order

Policies run **in order on the request** and **in inverse order on the response**. For example,
with X listed before Y, the request runs X then Y, and the response runs Y then X. Some policies
don't run on both phases. In Local Mode the order follows the `spec.policies` list. In Connected
Mode you reorder policies in API Manager, the same way as for included policies.

## Common mistakes

| Mistake | Fix |
| --- | --- |
| Guessing the name (`<name>-flex`, the crate name, the gcl.yaml name) | `make show-policy-ref-name` / `target/policy-ref-name.txt` |
| Copying the files flat into `config/`, or copying the `.wasm` | Put the two YAML files in `playground/config/custom-policies/` |
| `policies:` nested under a route or `services` | Put it under `spec`, next to `address` |
| Policy under test not first in the list | `make run` patches only `spec.policies[0]` |
| Leaving out `namespace: default` in the inline entry | Add it. It matches the gcl.yaml metadata namespace |
| Editing `api.yaml` and expecting Connected Mode to change | Connected Mode is configured in API Manager |

## See Also

- `pdk-test-locally`, `pdk-debug-local`: running the playground and testing with curl
- `pdk-publish-policies`: getting the policy into Exchange for Connected Mode
- `pdk-schema-definition`: the `gcl.yaml` properties that become `config`

## Source Ref

- Docs: https://docs.mulesoft.com/pdk/latest/policies-pdk-apply-policies
- Related: https://docs.mulesoft.com/gateway/latest/local-configuration-reference-guide,
  https://docs.mulesoft.com/gateway/latest/policies-reorder
- **Snapshot:** 2026-11-24
