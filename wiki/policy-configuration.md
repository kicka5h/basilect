# Policy Configuration

basilect enforces OPA policies on every `terraform plan` before apply. The
platform team defines the default enforcement scope (which environments each
policy runs in). Product teams can **tighten** enforcement for their repos —
extending policies to additional environments — but cannot loosen or disable
platform-mandated policies.

---

## How policy enforcement works

```
Product team pushes a change
        │
        ▼
tf-plan.yml / tf-deploy.yml
  │
  ├── terraform plan → tfplan.json
  ├── Merge policy data:
  │     policy/policy_defaults.json  (platform defaults)
  │   + basilect.yaml                (repo overrides, if present)
  │   + {"environment": "<env>"}    (current environment)
  │
  └── conftest test tfplan.json
        --policy policy/
        --data policy_defaults.json
        --data basilect.yaml
        --data env.json
              │
              ▼
        Each policy checks:
          1. Is this policy active for this environment?
          2. If yes, evaluate the rule
          3. If denied → block the PR / fail the deploy
```

---

## Default enforcement

Every policy has a default set of environments where it is enforced. These
defaults are defined by the platform team in `policy/policy_defaults.json`
and cannot be reduced by product teams.

| Policy | Default Environments | Configurable |
|--------|---------------------|-------------|
| No Public Resources | dev, staging, prod | No |
| No Network Modifications | dev, staging, prod | No |
| No RBAC Changes | dev, staging, prod | No |
| Require Storage Encryption | staging, prod | No |
| No Deletions Outside Dev | staging, prod | No |
| SKU Allowlist | dev, staging, prod | Yes |
| Required Tags | dev, staging, prod | Yes |
| AKS Admin Group Allowlist | dev, staging, prod | Yes |
| PostgreSQL Parameter Safety | dev, staging, prod | Yes |
| Require HA in Production | prod | No |
| Require Geo-Redundant Backups | prod | No |
| Kubernetes Version Pinning | staging, prod | Yes |
| Node Pool Size Limits | dev, staging, prod | Yes |
| Require Diagnostic Logging | staging, prod | No |
| CIDR Range Restrictions | dev, staging, prod | No |
| No Dangerous Provisioners | dev, staging, prod | No |
| Resource Naming Convention | dev, staging, prod | Yes |

View the full catalog with descriptions on the
[Dashboard Policies page](/policies).

---

## Customizing enforcement per repo

To customize which environments enforce which policies, create a
`basilect.yaml` file in the root of your infrastructure repo.

### Adding enforcement to more environments

The most common use case: a team wants a policy that normally only runs in
prod to also run in staging.

```yaml
# basilect.yaml
policy_config:
  overrides:
    # Enforce HA requirements in staging too (default: prod only)
    require_ha_in_prod:
      environments: [staging, prod]

    # Enforce geo-redundant backups in staging too
    require_geo_backup_in_prod:
      environments: [staging, prod]
```

### Enforcing version pinning in all environments

By default, Kubernetes version pinning only runs in staging and prod (so dev
can experiment). To enforce it everywhere:

```yaml
policy_config:
  overrides:
    deny_latest_k8s_version:
      environments: [dev, staging, prod]
```

### Tightening node pool limits

Override the per-environment node limits for your team:

```yaml
policy_config:
  overrides:
    limit_node_pool_max:
      environments: [dev, staging, prod]
      config:
        max_nodes:
          dev: 3
          staging: 10
          prod: 30
```

---

## What you cannot override

The following policies are **non-configurable** — they are enforced in their
default environments regardless of `basilect.yaml`:

- **No Public Resources** — always enforced everywhere
- **No Network Modifications** — always enforced everywhere
- **No RBAC Changes** — always enforced everywhere
- **Require Storage Encryption** — always in staging/prod
- **No Deletions Outside Dev** — always in staging/prod
- **Require HA in Production** — always in prod (can extend to staging, cannot remove from prod)
- **Require Diagnostic Logging** — always in staging/prod
- **CIDR Range Restrictions** — always enforced everywhere
- **No Dangerous Provisioners** — always enforced everywhere (blocks `null_resource`, `terraform_data`, `local-exec`, `remote-exec`)

Attempting to reduce the environment scope of a non-configurable policy in
`basilect.yaml` has no effect — the platform defaults always win.

---

## Environment aliases

The policy system recognizes environment aliases and resolves them to canonical
names before evaluating any policy. This means teams using non-standard
environment names still get correct policy enforcement.

| Alias | Resolves to |
|-------|-------------|
| `production` | `prod` |
| `uat` | `staging` |
| `test` | `dev` |
| `development` | `dev` |

Custom aliases can be added in `policy_defaults.json` under the
`environment_aliases` key, or per-repo in `basilect.yaml` under
`layout.custom_env_aliases`.

---

## Testing policies locally

Run the full policy suite against a local Terraform plan:

```bash
# Generate a plan
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Run policies with your overrides
conftest test tfplan.json \
  --policy policy/ \
  --data policy/policy_defaults.json \
  --data basilect.yaml \
  --data <(printf '{"environment":"dev","project":"myapp"}')
```

To test a specific policy:

```bash
conftest test tfplan.json \
  --policy policy/require_ha_in_prod.rego \
  --data <(printf '{"environment":"prod"}')
```

To run the OPA test suite:

```bash
opa test policy/ -v
```

---

## Adding a new policy

Platform engineers can add new policies by:

1. Create a `.rego` file in `policy/` following the naming convention
   `deny_<what>.rego` or `require_<what>.rego`
2. Create a matching `_test.rego` file
3. Add the policy to `policy/policies.json` with metadata
4. Add the default enforcement to `policy/policy_defaults.json`
5. Run `opa test policy/ -v` to verify
6. The dashboard picks up new policies automatically from the catalog

### Policy template

```rego
package basilect.deny_example_resource

import future.keywords.in
import data.basilect.lib

deny[msg] {
  # Use lib.policy_active to respect overrides and environment aliases
  lib.policy_active("deny_example_resource")

  change := input.resource_changes[_]
  change.change.actions[_] in {"create", "update"}
  change.type == "azurerm_example_resource"

  # Your enforcement logic here
  after := change.change.after
  after.some_dangerous_setting == true

  msg := sprintf(
    "[deny_example_resource] %s: some_dangerous_setting must be false.",
    [change.address],
  )
}
```

### Test template

```rego
package basilect.deny_example_resource_test

import future.keywords.in
import data.basilect.deny_example_resource

test_deny_dangerous_setting_in_prod {
  msgs := deny_example_resource.deny with input as {"resource_changes": [{
    "address": "azurerm_example_resource.this",
    "type": "azurerm_example_resource",
    "change": {"actions": ["create"], "after": {"some_dangerous_setting": true}},
  }]} with data.environment as "prod"
  count(msgs) == 1
}

test_allow_safe_setting_in_prod {
  msgs := deny_example_resource.deny with input as {"resource_changes": [{
    "address": "azurerm_example_resource.this",
    "type": "azurerm_example_resource",
    "change": {"actions": ["create"], "after": {"some_dangerous_setting": false}},
  }]} with data.environment as "prod"
  count(msgs) == 0
}
```

---

## Viewing enforcement from the dashboard

The [Policies page](/policies) shows:

- **Policy Catalog** tab — all available policies with severity, category,
  environments, and whether they're configurable
- **Violations** tab — live policy violations detected from CI workflow runs,
  filterable by repo and policy name

The [Compliance page](/compliance) shows whether each product repo is using
the construct library (which embeds policy enforcement) and whether it's on
the latest version.

The [Activity feed](/activity) shows policy violation events alongside deploys
and drift alerts in a unified timeline.
