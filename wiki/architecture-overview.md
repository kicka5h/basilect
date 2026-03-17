# Architecture Overview

## System layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Product team                                                    │
│  Writes a CDKTF stack in Python / TypeScript / C# / Java / Go  │
│  using the platform-maintained construct library                │
└─────────────────────────┬───────────────────────────────────────┘
                           │  git push → pull request / merge
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  infra.yml  (thin calling workflow in each product repo)        │
│  Delegates to nautilus/reusable-workflows:                         │
│                                                                  │
│  tf-validate  →  fmt-check + init -backend=false + validate     │
│  tf-changes   →  detect which env folders changed (PR only)     │
│  tf-plan      →  init + plan + OPA check + post PR comment      │
│                  runs only for affected environments, parallel   │
│  [gate job]   →  environment: stage / prod  (approval gate)     │
│  tf-deploy    →  init + plan + OPA check + apply + artifact     │
│                  dev → qa → stage → prod  (sequential)          │
│  tf-drift     →  daily: plan -detailed-exitcode + open issue    │
└─────────────────────────┬───────────────────────────────────────┘
                           │  module source via SSH deploy key
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  terraform-modules  (nautilus/terraform-modules — private)         │
│                                                                  │
│  modules/networking          modules/database/postgres           │
│  modules/compute/aks         governance/                        │
│                                                                  │
│  Enforces: naming conventions, required tags, security          │
│  defaults, HA requirements, management locks in staging/prod    │
└─────────────────────────┬───────────────────────────────────────┘
                           │  provisions resources
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Azure                                                           │
│  Remote state: Blob Storage (platform-managed, one file per     │
│    project+environment combination)                             │
│  Resources: VNet, AKS, PostgreSQL, etc.                         │
│  Azure Policy: deny public resources, restrict network/RBAC     │
│    writes in staging and prod                                    │
│  Management locks: CanNotDelete on all non-dev resources        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Responsibility split

| Concern | Product team | Platform team |
|---------|-------------|---------------|
| Which resources to create | ✅ | |
| Resource sizing (within approved SKU list) | ✅ | |
| Environment-specific configuration (tfvars) | ✅ | |
| Terraform module implementation | | ✅ |
| Security and compliance policy | | ✅ |
| Required resource tagging | | ✅ (injected automatically) |
| Remote state backend | | ✅ |
| Azure provider and OIDC credentials | | ✅ |
| CI/CD pipeline template | | ✅ |
| Construct libraries (auto-generated via codegen) | | ✅ |
| Dashboard (repo health, compliance, versions) | | ✅ |
| Nautilus GitHub App (bot identity) | | ✅ |
| OPA/conftest policy rules (`policy/`) | | ✅ |
| Azure Policy definitions and assignments | | ✅ |
| Reusable GitHub Actions workflows | | ✅ |
| Management lock removal for decommissions | | ✅ |
| Running `terraform apply` | | ✅ (pipeline only) |

Product teams are deliberately shielded from Terraform, state, and cloud-level
access controls. The platform team owns every enforcement layer beneath the
construct API.

---

## Enforcement layers

Three independent layers enforce policy. Each catches violations at a different
point, so no single bypass defeats all enforcement.

| Layer | Where | When caught | Who sees it |
|-------|-------|-------------|-------------|
| OPA/conftest (`policy/`) | CI/CD pipeline | After `terraform plan`, before apply | Product team — PR annotation |
| Azure Policy (`tf-modules/governance/`) | Azure ARM API | At `terraform apply` time | Product team — apply failure |
| Management locks (inside each TF module) | Azure ARM API | When deletion is attempted | Anyone — portal / CLI / Terraform |

### Rules enforced

| Rule | Environments |
|------|-------------|
| No public IPs or `public_network_access_enabled = true` | All |
| No `allow_blob_public_access = true` on storage accounts | All |
| No creation of network resources (VNets, subnets, NSGs, DNS zones, private endpoints) | Staging, prod |
| No RBAC changes (role assignments, role definitions, managed identities) | Staging, prod |
| No resource deletion or replacement | Staging, prod |
| VM and PostgreSQL SKUs must be on the platform-approved allowlist | All |
| Required tags (`managed_by`, `project`, `environment`) must be present | All |
| No `null_resource`, `terraform_data`, `local-exec`, or `remote-exec` | All |

Network infrastructure in staging and prod is centrally managed by the platform
team. Product teams reference existing subnet and DNS zone IDs via stack inputs —
they do not provision network resources there.

---

## Authentication

All Azure authentication uses OIDC (workload identity federation). No static
credentials are stored in GitHub secrets.

```
GitHub Actions runner
│
├── OIDC token  (id-token: write permission on the workflow)
│   → exchanged with Azure AD for a short-lived access token
│   → authenticates the Terraform AzureRM provider via ARM_USE_OIDC=true
│   → ARM_CLIENT_ID and ARM_SUBSCRIPTION_ID are stored as repo secrets
│   → ARM_TENANT_ID is stored as a repo variable (shared across envs)
│
└── TF_MODULES_DEPLOY_KEY  (SSH deploy key, read-only, per product repo)
    → loaded by webfactory/ssh-agent before `terraform init`
    → allows Terraform to clone git::ssh://git@github.com/nautilus/terraform-modules.git
```

Each product team gets a service principal per environment with a federated
credential scoped to `repo:<org>/<repo>:environment:<env>`. The platform team
provisions these during onboarding.

---

## State isolation

Every project + environment combination gets its own state file, keyed
automatically by `BaseAzureStack` from the `project` and `environment` arguments.
Product teams cannot change the key. Environment aliases (`production`, `uat`,
`test`, `development`) are resolved to their canonical form (`prod`, `staging`,
`dev`) before the state key is computed, so `production` and `prod` share the
same state file.

```
Azure Blob Storage account: platformtfstate
  container: tfstate
    portal/dev/terraform.tfstate
    portal/staging/terraform.tfstate
    portal/prod/terraform.tfstate
    myapp/dev/terraform.tfstate
    myapp/staging/terraform.tfstate
    myapp/prod/terraform.tfstate
    ...
```

State is locked via Azure blob lease during apply. If a pipeline job is cancelled
mid-run, the lease may persist. See
[Product Team Maintenance — State lock](platform-product-maintenance.md#state-lock)
for how to release it.

---

## Module versioning contract

The construct libraries pin an explicit Git tag when sourcing each Terraform module:

```
git::ssh://git@github.com/nautilus/terraform-modules.git//modules/networking?ref=v1.4.0
```

The construct library package version and the module tag it references are always
identical. Releasing `nautilus-infra==1.5.0` requires cutting `v1.5.0` in the module
repo first and updating the source strings in all five language libraries.

Product teams upgrade by changing one line in their dependency file. The module
pin upgrades automatically with the package.

---

## Data flow — pull request

```
Product team pushes a branch
        │
        ▼
[validate]              tf-validate.yml
  terraform fmt -check -recursive
  terraform init -backend=false
  terraform validate
        │
        ▼
[changes]               tf-changes.yml
  dorny/paths-filter → which of shared/dev/qa/stage/prod changed?
  outputs: shared, dev, qa, stage, prod  (true/false each)
        │
        ├─ shared OR dev   ──►  [plan-dev]    ─┐
        ├─ shared OR qa    ──►  [plan-qa]     ─┤  tf-plan.yml
        ├─ shared OR stage ──►  [plan-stage]  ─┤  parallel, cancel-in-progress
        └─ shared OR prod  ──►  [plan-prod]   ─┘
                                      │
                          each plan job:
                            terraform init -backend-config=../<env>/backend.hcl
                            terraform plan -var-file=../<env>/terraform.tfvars
                            conftest test  → policy gate (blocks PR if violated)
                            → plan summary posted as PR comment
```

## Data flow — merge to main

```
[deploy-dev]    tf-deploy.yml  (automatic)
[deploy-qa]     tf-deploy.yml  (automatic, after dev succeeds)
[gate-stage]    inline job, environment: stage  → approval required
[deploy-stage]  tf-deploy.yml  (after approval)
[gate-prod]     inline job, environment: prod   → approval required
[deploy-prod]   tf-deploy.yml  (after approval)
```

The gate jobs (`gate-stage`, `gate-prod`) are inline in the calling workflow, not
in the reusable workflows. This is required because GitHub evaluates environment
protection rules against the repository where the workflow job is defined. A job
inside a reusable workflow is evaluated against the reusable workflow's repo, which
does not have the product team's environment configuration.

## Code generation flow

```
Platform engineer tags terraform-modules (e.g., v1.5.0)
        │
        │  repository_dispatch: module-release
        ▼
codegen workflow  (project-nautilus)
  │
  ├── Generate Nautilus App token
  ├── Clone terraform-modules at the tag (repo configurable via MODULES_REPO)
  ├── Run nautilus-codegen:
  │     reads variables.tf + outputs.tf + module.json
  │     classifies variables (standard / platform / config / tags)
  │     handles nested types, tuples, heredocs, keyword conflicts
  │     annotates sensitive variables in generated code
  │     emits construct code for 5 languages
  │     (supports --manifest for multi-source module repos)
  └── Open PR as nautilus[bot]
        │
        │  PR triggers CI (because App token, not GITHUB_TOKEN)
        │  merge + tag → publish to package registries
        ▼
Product teams: bump dependency version → done
```

---

## Dashboard data flow

```
Browser → Next.js (Azure Static Web Apps)
            │
            ├── NextAuth.js  (Azure AD + GitHub OAuth)
            └── API routes   (server-side, cached)
                  │
                  └── GitHub API via Nautilus App (with throttling plugin)
                        ├── Paginated repo listing (handles >100 repos)
                        ├── Filter: archived repos excluded
                        ├── Workflow runs  → deploy status per environment
                        ├── Issues (drift label) → drift alerts
                        ├── Failed PR runs → policy violations
                        ├── Package manifests → construct versions
                        ├── Inline source scan → forked construct detection
                        └── Open PRs per repo
```

No database — all data is fetched from GitHub on demand with server-side TTL
caching (5 min for repos, 2 min for environments, 1 hour for versions).
GitHub App credentials are validated on save via `apps.getAuthenticated()`
before being persisted. Rate limits are handled automatically via
`@octokit/plugin-throttling` with retry on 429 responses.

---

## Data flow — scheduled drift detection

```
[drift-dev / drift-qa / drift-stage / drift-prod]   tf-drift.yml  (all parallel)
  terraform init
  terraform plan -detailed-exitcode
    exit 0 → no drift, job succeeds silently
    exit 2 → drift detected:
               check for existing open issue labelled "drift" + "<env>"
               if exists → add comment with new plan output
               if not    → open new issue with plan output
    exit 1 → plan error, job fails
```

The validate job does not run on the drift schedule — drift jobs authenticate
independently and a validate pass is not needed.
