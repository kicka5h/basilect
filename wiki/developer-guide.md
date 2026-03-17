# Developer Interface Reference

This document describes the interface Nautilus exposes to product teams. It covers
what they can build, what the platform enforces, and how to support them when
things go wrong. Share this document with product teams during onboarding.

---

## What product teams control

Product teams write a single CDKTF stack file in their language of choice. They
control:

- Which app-layer constructs to use (database, AKS) and how to configure them
- Resource sizes, within the platform-approved SKU list
- Environment-specific values (via tfvars)
- Stack outputs

Everything beneath the construct API — the networking layer, Terraform modules,
the pipeline, the state backend, the credentials, the OPA and Azure policies —
is owned by the platform team and not visible to product teams.

**Product teams do not manage any networking.** VNets, subnets, NSGs, and private
DNS zones are provisioned automatically by the platform layer inside
`BaseAzureStack`. App-layer constructs (database, AKS) wire to the correct
subnets and DNS zones automatically.

---

## Supported languages

| Language | Package | Registry |
|----------|---------|---------|
| Python | `nautilus-infra` | Internal PyPI |
| TypeScript | `@nautilus/infra` | Internal npm |
| C# | `Nautilus.Infra` | Internal NuGet |
| Java | `com.nautilus:infra` | Internal Maven |
| Go | `github.com/nautilus/infra-go` | Internal Go proxy |

All five produce identical Terraform JSON. The synthesized output is what the
pipeline operates on.

---

## What the platform enforces

### Blocked in all environments

| What | Policy |
|------|--------|
| Public IPs (`azurerm_public_ip`) | `deny_public_resources` |
| `public_network_access_enabled = true` on any resource | `deny_public_resources` |
| Public blob storage (`allow_blob_public_access = true`) | `deny_public_resources` |
| VM or PostgreSQL SKUs outside the approved list | `deny_budget_violations` |
| Missing required tags (`managed_by`, `project`, `environment`) | `deny_missing_required_tags` |
| Creating or modifying network resources (VNets, subnets, NSGs, DNS zones) | `deny_network_changes` |
| Creating or modifying RBAC (role assignments, role definitions, managed identities) | `deny_permission_changes` |
| Unsafe PostgreSQL server parameters | `deny_unsafe_postgres_params` |
| Unapproved AAD groups for AKS cluster-admin | `deny_unapproved_aks_admins` |
| `null_resource`, `terraform_data`, `local-exec`, `remote-exec` provisioners | `deny_dangerous_provisioners` |

### Blocked in staging and prod

| What | Policy |
|------|--------|
| Deleting or replacing any resource | `deny_deletions_outside_dev` |

### Approved SKU lists

**Virtual machines (AKS node pools)**

| Environment | Allowed SKUs |
|-------------|-------------|
| dev | `Standard_B2s`, `Standard_B4ms`, `Standard_D2s_v3`, `Standard_D4s_v3` |
| staging | dev SKUs + `Standard_D8s_v3`, `Standard_D16s_v3`, `Standard_E4s_v3`, `Standard_E8s_v3` |
| prod | `Standard_D4s_v3` through `Standard_D32s_v3`, `Standard_E4s_v3` through `Standard_E32s_v3`, `Standard_F8s_v2`, `Standard_F16s_v2` |

**PostgreSQL**

| Environment | Allowed SKUs |
|-------------|-------------|
| dev | `B_Standard_B1ms`, `B_Standard_B2ms`, `GP_Standard_D2s_v3` |
| staging | `GP_Standard_D2s_v3`, `GP_Standard_D4s_v3`, `GP_Standard_D8s_v3` |
| prod | `GP_Standard_D4s_v3` through `GP_Standard_D32s_v3`, `MO_Standard_E4ds_v4`, `MO_Standard_E8ds_v4` |

To add a SKU to the allowlist, update `policy/deny_budget_violations.rego`.

---

## Stack structure

All stacks inherit from `BaseAzureStack`. This base class:
- Wires up the AzureRM provider and remote state backend
- Injects required tags on every resource
- **Provisions the entire network layer automatically** (VNet, subnets, NSGs, private DNS zones)

Product teams must not configure the provider, backend, or any network resources
themselves.

```python
class MyStack(BaseAzureStack):
    def __init__(self, scope: Construct, id_: str):
        super().__init__(
            scope, id_,
            project="myproduct",           # lowercase, no spaces
            environment=os.environ["ENVIRONMENT"],  # dev | staging | prod
            resource_group="myproduct-dev-rg",
            location="eastus",             # optional, defaults to eastus
        )
```

The `project` and `environment` values are embedded in every resource name and in
the state key. Changing them after initial deployment destroys and recreates every
resource — treat them as permanent identifiers.

The stack accepts `dev`, `staging`, and `prod` as environment values. Common
aliases are also accepted and automatically resolved:

| Alias | Resolves to |
|-------|-------------|
| `production` | `prod` |
| `uat` | `staging` |
| `test` | `dev` |
| `development` | `dev` |

The stack raises `ValueError` at synthesis time if `environment` is not one of
the canonical values or a recognized alias.

### What the stack provides automatically

After calling `super().__init__()`, the following are available on `self`:

| Property | Type | Description |
|----------|------|-------------|
| `subnet_ids` | `dict[str, str]` | Map of subnet name to resource ID (`aks`, `postgres`, `endpoints`) |
| `dns_zone_ids` | `dict[str, str]` | Map of DNS zone FQDN to resource ID |

App-layer constructs wire to these automatically — developers do not need to
reference them directly.

---

## Available constructs

### DatabaseConstruct

Provisions a PostgreSQL Flexible Server with private VNet integration. The
construct automatically wires to the platform-managed `postgres` subnet and
private DNS zone.

```python
DatabaseConstruct(self, "db",
    stack=self,                         # required — provides network wiring
    project="portal",
    environment="dev",
    resource_group="portal-dev-rg",
    location="eastus",
    admin_password=os.environ["DB_ADMIN_PASSWORD"],
    config=PostgresConfig(
        databases=["appdb", "cachedb"],
        sku="GP_Standard_D2s_v3",
        pg_version="15",
    ),
)
```

**Key inputs — PostgresConfig**

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `databases` | `list[str]` | `[]` | Database names to create |
| `sku` | `str` | `GP_Standard_D2s_v3` | Must be on the approved list |
| `storage_mb` | `int` | `32768` | Min 32 GiB |
| `pg_version` | `str` | `"15"` | 14, 15, or 16 |
| `ha_enabled` | `bool` | `False` | Required `True` in prod |
| `geo_redundant` | `bool` | `False` | Enables geo-redundant backups |
| `server_configs` | `dict[str, str]` | `{}` | PostgreSQL parameter overrides (allowlisted only) |

The admin password must come from `os.environ["DB_ADMIN_PASSWORD"]`, which the
pipeline injects from the `DB_ADMIN_PASSWORD` GitHub secret. It must never be
hardcoded in the stack.

**Allowed server parameters**: `statement_timeout`, `lock_timeout`,
`idle_in_transaction_session_timeout`, `log_min_duration_statement`,
`log_statement`, `timezone`, `autovacuum_*`, `jit`, `search_path`, and others.
See `policy/deny_unsafe_postgres_params.rego` for the full allowlist. Dangerous
parameters (SSL, shared memory, WAL, extension loading) are blocked.

**Key outputs**

| Output | Notes |
|--------|-------|
| `fqdn` | Use as the application connection host |
| `server_id` | Use for downstream RBAC |
| `server_name` | |

---

### AksConstruct

Provisions an AKS cluster with Azure CNI, AAD RBAC, and Log Analytics monitoring.
The construct automatically wires to the platform-managed `aks` subnet.

```python
AksConstruct(self, "aks",
    stack=self,                         # required — provides network wiring
    project="portal",
    environment="dev",
    resource_group="portal-dev-rg",
    location="eastus",
    log_workspace_id=os.environ["LOG_WORKSPACE_ID"],
    config=AksConfig(
        kubernetes_version="1.29",
        system_node_count=3,
        additional_node_pools={
            "workers": NodePoolConfig(
                vm_size="Standard_D4s_v3",
                node_count=2,
                enable_auto_scaling=True,
                min_count=2,
                max_count=8,
            ),
        },
    ),
)
```

**Key inputs — AksConfig**

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `kubernetes_version` | `str` | `"1.29"` | Platform-approved versions only |
| `system_node_vm_size` | `str` | `Standard_D2s_v3` | |
| `system_node_count` | `int` | `3` | Use 3 for zone redundancy in prod |
| `additional_node_pools` | `dict` | `{}` | Pool name max 12 chars |
| `admin_group_object_ids` | `list[str]` | `[]` | Must be platform-approved AAD group IDs |
| `service_cidr` | `str` | `10.240.0.0/16` | Must not overlap VNet |
| `dns_service_ip` | `str` | `10.240.0.10` | Within `service_cidr` |

The system node pool is tainted `CriticalAddonsOnly=true:NoSchedule`. Product
teams should always add at least one additional pool for application workloads.

**Key inputs — NodePoolConfig**

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `vm_size` | `str` | `Standard_D4s_v3` | |
| `node_count` | `int` | `2` | Used when auto-scaling is off |
| `enable_auto_scaling` | `bool` | `False` | |
| `min_count` | `int` | `1` | Requires `enable_auto_scaling=True` |
| `max_count` | `int` | `10` | |
| `labels` | `dict` | `{}` | Kubernetes node labels |
| `taints` | `list[str]` | `[]` | e.g. `["dedicated=gpu:NoSchedule"]` |

**Key outputs**

| Output | Notes |
|--------|-------|
| `cluster_id` | Resource ID |
| `kubelet_identity_object_id` | Assign ACR pull, Key Vault read |
| `cluster_identity_principal_id` | Assign Network Contributor |

---

## Complete example

```python
import os
from nautilus_infra import BaseAzureStack
from nautilus_infra.constructs import (
    DatabaseConstruct, AksConstruct,
    PostgresConfig, AksConfig, NodePoolConfig,
)

class PortalStack(BaseAzureStack):
    def __init__(self, scope, id_):
        super().__init__(scope, id_,
            project="portal",
            environment=os.environ["ENVIRONMENT"],
            resource_group=f"portal-{os.environ['ENVIRONMENT']}-rg",
            location="eastus",
        )

        db = DatabaseConstruct(self, "db",
            stack=self,
            project="portal",
            environment=os.environ["ENVIRONMENT"],
            resource_group=f"portal-{os.environ['ENVIRONMENT']}-rg",
            location="eastus",
            admin_password=os.environ["DB_ADMIN_PASSWORD"],
            config=PostgresConfig(
                databases=["portaldb"],
                pg_version="15",
            ),
        )

        aks = AksConstruct(self, "aks",
            stack=self,
            project="portal",
            environment=os.environ["ENVIRONMENT"],
            resource_group=f"portal-{os.environ['ENVIRONMENT']}-rg",
            location="eastus",
            log_workspace_id=os.environ["LOG_WORKSPACE_ID"],
            config=AksConfig(
                additional_node_pools={
                    "workers": NodePoolConfig(
                        vm_size="Standard_D4s_v3",
                        enable_auto_scaling=True,
                        min_count=2,
                        max_count=8,
                    ),
                },
            ),
        )

        # Expose outputs for application configuration
        self.add_output("db_fqdn", db.fqdn)
        self.add_output("cluster_id", aks.cluster_id)
```

---

## Secrets and environment variables

No secrets are hardcoded in stack files. All sensitive values come from environment
variables, injected by the pipeline from GitHub Actions secrets.

| Secret | Injected as | Used for |
|--------|-------------|---------|
| `DB_ADMIN_PASSWORD` | `DB_ADMIN_PASSWORD` env var | PostgreSQL admin password |
| `LOG_WORKSPACE_ID` | `LOG_WORKSPACE_ID` env var | Log Analytics workspace resource ID |

For local synthesis, product teams export these manually from a secure source
(e.g. a key vault) before running `cdktf synth`. Synthesis is a dry-run — no
Azure resources are created.

---

## The pipeline

Product teams copy `infra.yml` from the reference implementation (`tf-azure/`) into
their repo. They update the stack name references and do not otherwise modify it.

**PR behaviour**
1. `validate` — fmt-check + validate, runs on every PR
2. `changes` — detects which environment folders changed
3. `plan-<env>` — runs for each affected environment in parallel; posts a plan
   comment; blocks the PR if any OPA policy is violated

**Merge to main**
1. `deploy-dev` → `deploy-qa` — automatic, sequential
2. `gate-stage` — requires manual approval from the platform team
3. `deploy-stage`
4. `gate-prod` — requires manual approval from the platform team
5. `deploy-prod`

Production applies require the platform team as required reviewer on the GitHub
`production` Environment. Configure this in the product team's repo settings.

---

## Stack outputs

Outputs appear in the CI/CD job summary and in the remote state for cross-stack
references. Product teams should expose all values their application configuration
needs (e.g. database FQDN, cluster ID).

---

## Nautilus Dashboard

The Nautilus Dashboard at your organization's dashboard URL provides real-time
visibility into your infrastructure repos:

- **Repository overview** — see deploy status, drift alerts, and open PRs across all repos (paginated, no 100-repo limit)
- **Environment health** — per-environment (dev/qa/staging/prod) deploy and drift status
- **Policy compliance** — OPA violations with weekly trends
- **Module versions** — check if your repo is on the latest construct version; repos with forked/inline construct code are flagged separately
- **Open PRs** — all open infrastructure PRs with CI status

Archived repos are automatically hidden from the dashboard.

To have your repo appear on the dashboard, add the `nautilus-managed` topic:

```bash
gh repo edit <owner>/<repo> --add-topic nautilus-managed
```

---

## Repo layout configuration

Teams with non-standard directory layouts can declare their structure in
`nautilus.yaml` at the repo root:

```yaml
layout:
  type: standard          # standard | flat | workspace | cdktf
  environments: [dev, staging, prod]
  custom_env_aliases:
    production: prod
    uat: staging
```

The `type` field tells the reusable workflows how to discover and plan each
environment. The `custom_env_aliases` are merged with the platform defaults
and used by both `BaseAzureStack` and the OPA policy engine.

---

## Upgrading the construct library

The platform team announces breaking changes with a migration guide at least two
weeks before release. Product teams upgrade by updating their pinned version in
their dependency file and running a local `cdktf synth` to verify compatibility.

Version ranges must not be used — product teams must always pin an exact version.

---

## Troubleshooting reference

This section covers the errors product teams will encounter. Use it to diagnose
issues quickly when supporting them.

### Synthesis fails — import or module not found

The construct library is not installed or was installed from the wrong registry.
Confirm the product team is using the correct internal registry URL and the pinned
version exists there.

### `ValueError: environment must be dev, staging, or prod`

The `ENVIRONMENT` environment variable is missing or has an unexpected value. The
accepted values are `dev`, `staging`, and `prod`, plus the aliases `production`,
`uat`, `test`, and `development`. Check how the pipeline injects it.

### OPA policy violation — `deny_dangerous_provisioners`

The product team is using `null_resource`, `terraform_data`, `local-exec`, or
`remote-exec` in their Terraform code. These resource types and provisioners
introduce unmanaged side effects that bypass policy enforcement. Replace them
with proper Terraform resources or move the logic to an application-layer tool.

### `terraform init` fails — `ssh: connect to host github.com port 22`

The `TF_MODULES_DEPLOY_KEY` secret is missing or invalid in the product team's
repo. Verify the secret exists. If it was recently rotated on the module repo,
ensure the updated private key was also added to the product repo.

### OPA policy violation — `deny_network_changes`

The product team is trying to create network resources directly. This is not
allowed — all networking is managed by the platform layer inside `BaseAzureStack`.
Remove any `NetworkConstruct` usage or direct network resource definitions from
the stack.

### OPA policy violation — `deny_unsafe_postgres_params`

The product team is setting a PostgreSQL server parameter that is not on the
approved list. Check `policy/deny_unsafe_postgres_params.rego` for the allowlist.
If the parameter is legitimately needed, the platform team can add it.

### OPA policy violation — `deny_unapproved_aks_admins`

The product team is using an AAD group ID for AKS cluster-admin that hasn't been
registered by the platform team. The platform team must verify the group and add
its object ID to the approved list in `policy/deny_unapproved_aks_admins.rego`.

### OPA policy violation — `deny_budget_violations`

The product team is using a VM or PostgreSQL SKU that's not on the approved list
for their target environment. Refer to the SKU tables above.

### `terraform plan` reports drift

The daily drift-detection workflow has opened a GitHub issue. The platform team
investigates whether the drift was caused by an out-of-band change (e.g. portal
click, Azure Policy remediation) and decides whether to reconcile by re-applying
or by importing the change into state.
