# Bootstrap

The `bootstrap/` directory contains Terraform that provisions the Azure and Azure
AD resources the platform depends on before any product team infrastructure can
be deployed. Run these once when first implementing Nautilus, then once per new
product team during onboarding.

```
bootstrap/
├── platform/        Run once — creates the state backend and platform SP
└── product-team/    Run once per product team — creates SPs, OIDC creds, and RBAC
```

---

## Prerequisites

Before running bootstrap you need:

- **Azure CLI** authenticated as a principal with:
  - Owner on the platform subscription (to create RBAC assignments)
  - Global Administrator or Application Administrator in Azure AD (to create applications and service principals)
- **Terraform ≥ 1.7** installed locally
- The `github_org` and `github_repo` for the Nautilus platform repo

---

## Step 1 — Platform bootstrap

Run this once to create the shared Azure infrastructure the entire platform depends on.

### What it creates

| Resource | Purpose |
|----------|---------|
| `azurerm_resource_group` | Container for platform infrastructure |
| `azurerm_storage_account` | Terraform state backend (`platformtfstate`) |
| `azurerm_storage_container` | `tfstate` container within the account |
| `azurerm_management_lock` | Protects the state account from accidental deletion |
| `azuread_application` + SP | Platform service principal (`nautilus-platform`) |
| `azuread_application_federated_identity_credential` | OIDC credential scoped to the Nautilus repo's main branch |
| `azurerm_role_assignment` × 2 | Contributor on platform subscription; Storage Blob Data Contributor on state account |

### Apply

```bash
cd bootstrap/platform

# Authenticate
az login
az account set --subscription <platform-subscription-id>

# Initialise with local state (intentional — this creates the remote backend)
terraform init

# Review the plan
terraform plan \
  -var="platform_subscription_id=<platform-subscription-id>" \
  -var="github_org=nautilus" \
  -var="github_repo=project-nautilus"

# Apply
terraform apply \
  -var="platform_subscription_id=<platform-subscription-id>" \
  -var="github_org=nautilus" \
  -var="github_repo=project-nautilus"
```

### Read the outputs

After apply, Terraform prints the `next_steps` output with ready-to-copy values:

```bash
terraform output next_steps
terraform output state_storage_account_id   # needed for product-team bootstrap
terraform output tenant_id                  # AZURE_TENANT_ID variable
```

Set the GitHub Actions secrets and variables on the Nautilus repo as instructed.

### Optional: migrate local state to the remote backend

The platform bootstrap uses local state by default. To store its own state
remotely (recommended for team environments):

```bash
# Add to bootstrap/platform/main.tf — replace the backend "local" block:
backend "azurerm" {
  storage_account_name = "platformtfstate"
  container_name       = "tfstate"
  key                  = "bootstrap/platform/terraform.tfstate"
  use_oidc             = true
}

terraform init -migrate-state
```

---

## Step 2 — Product team bootstrap

Run this once per product team when onboarding them. It provisions service
principals and OIDC federated credentials for all four environments.

### What it creates (per environment: dev, qa, stage, prod)

| Resource | Purpose |
|----------|---------|
| `azuread_application` + SP | One service principal per environment (`<product>-<env>`) |
| `azuread_application_federated_identity_credential` | OIDC credential scoped to `repo:<org>/<repo>:environment:<env>` |
| `azurerm_role_assignment` (Contributor) | Access to the environment's Azure subscription |
| `azurerm_role_assignment` (Storage Blob Data Contributor) | Access to the platform state account |

### Apply

```bash
cd bootstrap/product-team

terraform init \
  -backend-config="storage_account_name=platformtfstate" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=bootstrap/portal/terraform.tfstate" \
  -backend-config="use_oidc=true"

terraform plan \
  -var="product_name=portal" \
  -var="github_org=nautilus" \
  -var="github_repo=portal-infra" \
  -var='subscription_ids={"dev":"<dev-sub-id>","qa":"<qa-sub-id>","stage":"<stage-sub-id>","prod":"<prod-sub-id>"}' \
  -var="state_storage_account_id=<state_storage_account_id from platform output>" \
  -var="platform_subscription_id=<platform-subscription-id>"

terraform apply [same vars]
```

If all environments share a subscription, repeat the same ID for each key:

```bash
-var='subscription_ids={"dev":"<shared-sub-id>","qa":"<shared-sub-id>","stage":"<shared-sub-id>","prod":"<prod-sub-id>"}'
```

### Read the outputs

```bash
terraform output github_secrets
```

This prints the full set of GitHub Actions secrets to copy into the product team's
repo — one `CLIENT_ID` and `SUBSCRIPTION_ID` per environment, plus `TENANT_ID`.

### Set the remaining secrets manually

The following secrets contain sensitive values and are not managed by Terraform.
Set them via `gh secret set` or the GitHub UI:

```bash
# SSH deploy key for terraform-modules
gh secret set TF_MODULES_DEPLOY_KEY \
  --repo nautilus/portal-infra \
  --body "$(cat /path/to/tf_modules_key)"

# PostgreSQL admin password
gh secret set DB_ADMIN_PASSWORD \
  --repo nautilus/portal-infra \
  --body "$(openssl rand -base64 32)"

# Log Analytics workspace resource ID
gh secret set LOG_WORKSPACE_ID \
  --repo nautilus/portal-infra \
  --body "/subscriptions/.../workspaces/platform-logs"
```

See [Module Maintenance — Provisioning a deploy key](platform-module-maintenance.md#provisioning-a-deploy-key-for-a-new-product-team)
for SSH key generation.

---

## Running product-team bootstrap for multiple teams

Each product team gets its own Terraform state file keyed by product name.
Run the module separately for each team:

```bash
for product in portal myapp billing; do
  terraform -chdir=bootstrap/product-team init \
    -backend-config="key=bootstrap/${product}/terraform.tfstate" \
    [other backend-config flags]

  terraform -chdir=bootstrap/product-team apply \
    -var="product_name=${product}" \
    [other vars]
done
```

---

## Rerunning bootstrap

Both modules are idempotent — rerunning them against an existing deployment
produces no changes unless variables have changed. This makes them safe to rerun:

- To rotate OIDC federated credentials: change the `display_name` on the
  `azuread_application_federated_identity_credential` resource, apply, then
  update the credential subject if needed.
- To add a new product team environment: update the `subscription_ids` variable
  and apply — Terraform will create the new SP and RBAC without touching existing ones.
- To change a subscription assignment: update the relevant `subscription_ids` entry,
  apply — Terraform will update the role assignments.

---

## What bootstrap does not provision

| Item | Where to handle it |
|------|--------------------|
| The `platform-infra` GitHub team | Create manually in GitHub org settings |
| GitHub repository creation | `gh repo create` or GitHub UI |
| GitHub repo configuration (branch protection, environments, labels) | `setup/scripts/apply-repo-config.sh` — see [Repository Setup](repository-setup.md) |
| Internal package registries (PyPI, npm, NuGet, Maven, Go proxy) | Organisation-specific; choose Artifactory, Nexus, or GitHub Packages |
| Azure Policy governance module deployment | `cd tf-modules/governance && terraform apply` — see [Module Maintenance](platform-module-maintenance.md#azure-policy) |
| SSH deploy keys for terraform-modules | Generated and added manually — see [Module Maintenance](platform-module-maintenance.md#provisioning-a-deploy-key-for-a-new-product-team) |
