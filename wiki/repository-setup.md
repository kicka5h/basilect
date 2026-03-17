# Repository Setup

The `setup/` directory contains everything needed to configure a GitHub repository
for use with Nautilus — branch protection, GitHub Environments, team permissions,
labels, CODEOWNERS, issue templates, and PR templates. A single script and a
`workflow_dispatch` workflow apply the full configuration from one command.

---

## Directory layout

```
setup/
├── configs/
│   ├── terraform-modules.json    Platform module library
│   ├── reusable-workflows.json   Shared CI/CD workflows
│   ├── construct-library.json    Any of the five construct library repos
│   └── product-team.json         Any product team infrastructure repo
├── scripts/
│   └── apply-repo-config.sh      Applies a config using the gh CLI
└── templates/
    ├── CODEOWNERS.platform        For terraform-modules and reusable-workflows
    ├── CODEOWNERS.construct       For construct library repos
    ├── CODEOWNERS.product         For product team repos
    ├── pull_request_template.platform.md
    ├── pull_request_template.product.md
    └── issue_templates/
        ├── bug_report.yml         All platform repos
        ├── new_module.yml         terraform-modules only
        ├── sku_request.yml        terraform-modules only
        └── infra_request.yml      Product team repos
```

---

## What each config applies

| Config | Repo settings | Branch protection | Environments | Teams | Labels | Templates |
|--------|-------------|-------------------|-------------|-------|--------|-----------|
| `terraform-modules` | Private, squash-only | 2 reviews, code owners, CI required | — | `platform-infra` = maintain | 10 labels | CODEOWNERS + PR + 3 issue templates |
| `reusable-workflows` | Private, squash-only | 2 reviews, code owners, CI required | — | `platform-infra` = maintain | 8 labels | CODEOWNERS + PR + bug report |
| `construct-library` | Private, squash-only | 1 review, code owners, CI required | `registry` (team approval) | `platform-infra` = maintain | 3 labels | CODEOWNERS + PR + bug report |
| `product-team` | Private, squash-only | 1 review, code owners, validate required | `stage` + `prod` (team approval, 10-min wait) | `platform-infra` = push | 5 labels | CODEOWNERS + PR + infra request |

### Branch protection details

| Config | Required reviews | Dismiss stale | Code owner review | Enforce admins |
|--------|-----------------|--------------|-------------------|---------------|
| `terraform-modules` | 2 | ✅ | ✅ | ✅ |
| `reusable-workflows` | 2 | ✅ | ✅ | ✅ |
| `construct-library` | 1 | ✅ | ✅ | ✅ |
| `product-team` | 1 | ✅ | ✅ | ❌ |

Required status check contexts (must match CI job IDs exactly):

| Config | Contexts |
|--------|---------|
| `terraform-modules` | `validate`, `test` |
| `reusable-workflows` | `test` |
| `construct-library` | `test` |
| `product-team` | `validate` |

---

## Prerequisites

Before running setup, you need:

1. **A PAT with the right scopes.** Create a Personal Access Token (classic) with:
   - `repo` — read/write repository settings, branch protection, environments
   - `admin:org` — manage team repository permissions
   - `read:org` — resolve team slugs to IDs

   Store it as `PLATFORM_GITHUB_TOKEN` in this repo's Actions secrets.

2. **The target repository must already exist.** The script configures an existing
   repo — it does not create one. Create the repo first via the GitHub UI or
   `gh repo create`.

3. **The `platform-infra` team must exist** in the target organization. The script
   resolves the team slug to an ID when configuring environments and team permissions.

4. **`jq` and `gh` CLI** installed if running the script locally.

---

## Running via GitHub Actions (recommended)

1. Go to **Actions → Repository setup → Run workflow**
2. Fill in the inputs:
   - **org**: your GitHub organization slug
   - **repo**: the repository name (e.g. `portal-infra`)
   - **config**: choose from the dropdown
   - **dry run**: check this first to preview changes without applying them
3. Click **Run workflow**

The workflow uses the `PLATFORM_GITHUB_TOKEN` secret. The job output shows each
step and flags any errors.

---

## Running locally

```bash
# Authenticate the gh CLI with a PAT that has repo + admin:org + read:org scopes
export GH_TOKEN=ghp_your_token_here

# Dry run first — see what will happen without making changes
bash setup/scripts/apply-repo-config.sh nautilus portal-infra setup/configs/product-team.json --dry-run

# Apply for real
bash setup/scripts/apply-repo-config.sh nautilus portal-infra setup/configs/product-team.json
```

---

## Onboarding a new product team repo

Run the `product-team` config against the new repo, then complete the
remaining secrets manually (these cannot be set via the setup script because
they contain sensitive values):

```bash
bash setup/scripts/apply-repo-config.sh nautilus portal-infra setup/configs/product-team.json
```

After the script completes, add these secrets to the product team's repo manually:

| Secret | Scope | Notes |
|--------|-------|-------|
| `DEV_AZURE_CLIENT_ID` | Actions | OIDC SP client ID for dev |
| `DEV_AZURE_SUBSCRIPTION_ID` | Actions | Azure subscription ID for dev |
| `QA_AZURE_CLIENT_ID` | Actions | OIDC SP client ID for QA |
| `QA_AZURE_SUBSCRIPTION_ID` | Actions | Azure subscription ID for QA |
| `STAGE_AZURE_CLIENT_ID` | Actions | OIDC SP client ID for staging |
| `STAGE_AZURE_SUBSCRIPTION_ID` | Actions | Azure subscription ID for staging |
| `PROD_AZURE_CLIENT_ID` | Actions | OIDC SP client ID for prod |
| `PROD_AZURE_SUBSCRIPTION_ID` | Actions | Azure subscription ID for prod |
| `AZURE_TENANT_ID` | Actions variable | Shared across all environments |
| `TF_MODULES_DEPLOY_KEY` | Actions | Private half of the SSH deploy key |
| `DB_ADMIN_PASSWORD` | Actions | PostgreSQL admin password |
| `LOG_WORKSPACE_ID` | Actions | Log Analytics workspace resource ID |

See [Product Team Maintenance — Onboarding](platform-product-maintenance.md#onboarding-a-new-product-team)
for the full onboarding checklist, including OIDC federated credential configuration.

---

## Setting up a construct library repo

Run the `construct-library` config, then add the `REGISTRY_TOKEN` secret scoped
to the `registry` environment (not at the repo level):

```bash
bash setup/scripts/apply-repo-config.sh nautilus nautilus-infra-python setup/configs/construct-library.json
```

Then in the repo: **Settings → Environments → registry → Add secret → `REGISTRY_TOKEN`**

The `REGISTRY_TOKEN` is the credential used by the `publish` job to push packages
to the internal registry. Scoping it to the environment ensures it is only
accessible after a platform team reviewer approves the publish job.

---

## Customising a config

The JSON configs are templates. For one-off customisations (e.g. a different repo
description, an extra label, or an additional reviewer team), edit a copy of the
relevant config before running the script — do not modify the canonical configs in
`setup/configs/` unless the change should apply to all future repos of that type.

To add a new approved SKU label to a specific product team repo after initial
setup, run the script again with an updated config — it will create new labels and
update existing ones without touching other settings.

---

## What the script does not manage

The following require manual action or a separate provisioning step:

- **Creating the repository** — use `gh repo create` or the GitHub UI first
- **GitHub Actions secrets** — secrets cannot be set via the `gh api` without
  encryption; use `gh secret set` or the UI
- **OIDC federated credentials on service principals** — set via `az ad app federated-credential create`
- **Azure RBAC assignments** — handled by the platform team's Azure provisioning steps
- **SSH deploy key generation** — see [Module Maintenance — Access control](platform-module-maintenance.md#access-control)
