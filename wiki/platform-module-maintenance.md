# Module Maintenance

How the platform team manages `nautilus/terraform-modules` — the private Terraform
module library that backs every Nautilus construct.

---

## Repository conventions

### Directory layout

```
terraform-modules/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── tests/networking.tftest.hcl
│   │   └── README.md
│   ├── database/
│   │   └── postgres/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── tests/database.tftest.hcl
│   │       └── README.md
│   └── compute/
│       └── aks/
│           ├── main.tf
│           ├── variables.tf
│           ├── outputs.tf
│           ├── tests/aks.tftest.hcl
│           └── README.md
├── governance/
├── CHANGELOG.md
└── .github/workflows/ci.yml
```

Each module is self-contained. Modules do not call each other — composition happens
in the CDKTF constructs.

### File responsibilities

| File | Purpose |
|------|---------|
| `variables.tf` | All input declarations with descriptions and validations |
| `main.tf` | All resources. No `variable` or `output` blocks. |
| `outputs.tf` | All outputs with descriptions |
| `tests/*.tftest.hcl` | `terraform test` suite using `mock_provider` |
| `README.md` | Usage example, inputs table, outputs table |

All resource logic stays in `main.tf`. Do not split resources across multiple
`.tf` files — it makes code review and grep harder.

---

## Variable conventions

Every module declares these standard variables:

```hcl
variable "project" {
  description = "Short project name. Used in resource naming and tags."
  type        = string
}

variable "environment" {
  description = "Deployment environment: dev, staging, or prod."
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "resource_group_name" {
  description = "Name of the Azure resource group."
  type        = string
}

variable "location" {
  description = "Azure region."
  type        = string
  default     = "eastus"
}

variable "tags" {
  description = "Azure resource tags. Required tags are injected by the construct library."
  type        = map(string)
  default     = {}
}
```

Rules:
- Every variable must have a `description`.
- Enumerations must have a `validation` block — fail at plan time, not apply.
- Sensitive values (passwords, keys) must be marked `sensitive = true`.
- Optional variables must have a `default`. Required variables must not.
- Use `optional(type, default)` inside `object()` types to avoid forcing callers
  to specify every field.

### Naming resources

Use a consistent `local` prefix and the `this` alias for each module's primary resource:

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
}

resource "azurerm_virtual_network" "this" {
  name = "${local.name_prefix}-vnet"
  ...
}
```

The `this` alias keeps output references readable (`azurerm_virtual_network.this.id`).

---

## Output conventions

Every output must have a `description`. Sensitive outputs must be marked
`sensitive = true`.

```hcl
output "server_id" {
  description = "Resource ID of the PostgreSQL Flexible Server."
  value       = azurerm_postgresql_flexible_server.this.id
}

output "administrator_password" {
  description = "Admin password. Use to bootstrap connection strings."
  value       = azurerm_postgresql_flexible_server.this.administrator_password
  sensitive   = true
}
```

Output names must exactly match the `get_string()` calls in the corresponding
CDKTF construct. Renaming an output is a breaking change — it requires updating
all five language libraries in the same release and a major version bump.

---

## Adding a new module

### 1. Open a design issue

Before writing any code, document:
- What resource(s) the module will manage
- What variables it will expose
- What outputs it will produce
- Why an existing module cannot cover the use case

Get at least one other platform engineer to review the design before starting.

### 2. Create the module directory

```bash
mkdir -p modules/<category>/<resource-name>
touch modules/<category>/<resource-name>/{main.tf,variables.tf,outputs.tf,README.md}
mkdir -p modules/<category>/<resource-name>/tests
```

### 3. Write variables first

Define inputs before resources. This forces clarity about the module's contract
before implementation details are committed to.

### 4. Implement main.tf

Follow the naming and aliasing conventions above. Add `lifecycle` blocks where
appropriate:

```hcl
lifecycle {
  # Passwords are rotated outside Terraform.
  ignore_changes = [administrator_password]
}
```

Add a `azurerm_management_lock` resource for non-dev environments:

```hcl
resource "azurerm_management_lock" "this" {
  count      = var.environment != "dev" ? 1 : 0
  name       = "${local.name_prefix}-lock"
  scope      = azurerm_<resource>.this.id
  lock_level = "CanNotDelete"
  notes      = "Managed by Nautilus. Remove via supervised decommission only."
}
```

### 5. Write outputs

Include all values a downstream construct might reasonably need: resource IDs,
FQDNs, managed identity principal IDs. Outputs are cheap; omitting one later is
a breaking change.

### 6. Write tests

Create a `tests/<module>.tftest.hcl` file using `mock_provider "azurerm" {}`.
Test at minimum: naming conventions, resource counts, management lock absent in dev
and present in staging/prod, and all declared outputs.

```hcl
mock_provider "azurerm" {}

variables {
  project            = "test"
  environment        = "dev"
  resource_group_name = "test-rg"
  location           = "eastus"
}

run "naming_convention" {
  assert {
    condition     = azurerm_<resource>.this.name == "test-dev-<suffix>"
    error_message = "Resource name does not follow the naming convention."
  }
}
```

### 7. Add a `module.json` for the codegen

Create a `module.json` alongside `variables.tf` and `outputs.tf`. This tells the
code generator how to map the Terraform module to a construct:

```json
{
  "constructName": "MyResourceConstruct",
  "moduleId": "my-resource",
  "modulePath": "modules/category/my-resource",
  "moduleRef": "v1.5.0",
  "repoUrl": "git::ssh://git@github.com/nautilus/terraform-modules.git",
  "platformInputs": {
    "subnet_id": "subnetIds.my-resource"
  },
  "propAliases": {
    "resource_group_name": "resourceGroup"
  }
}
```

**Key fields:**

| Field | Purpose |
|-------|---------|
| `platformInputs` | Variables resolved from `BaseAzureStack` (hidden from developers) |
| `propAliases` | Override the auto-generated prop name for a variable |
| `transforms` | Type conversions (e.g., `bool` → `"ZoneRedundant"/"Disabled"`) |

The codegen reads `variables.tf` and `outputs.tf` directly — it automatically
classifies each variable as standard, required, config (optional with default),
platform, or tags based on the HCL definitions and `module.json`.

### 8. Generate constructs

Run the codegen to generate construct code for all five languages:

```bash
cd codegen
npx tsx src/cli.ts generate \
  --modules-dir ../terraform-modules/modules \
  --output-dir ../constructs/cdktf
```

Or trigger the codegen workflow, which opens a PR as `nautilus[bot]`:

```bash
gh workflow run codegen.yml -f module_ref=v1.5.0
```

The codegen produces idiomatic code for each language: TypeScript interfaces,
Python dataclasses, Java builders, C# records, and Go structs — all from the
same module schema.

The codegen correctly handles:
- **Nested types**: Maps and lists containing nested objects are parsed with
  depth-tracking (not naive comma splitting)
- **Tuple types**: `tuple(string, number, bool)` maps to `unknown[]` (TS),
  `list` (Python), `List<Object>` (Java/C#), `[]interface{}` (Go)
- **Heredoc defaults**: `<<EOF ... EOF` default values are extracted correctly
- **Keyword conflicts**: Generated field names that collide with language
  reserved words (`class`, `type`, `import`, etc.) are suffixed with `_`
- **Sensitive annotations**: Variables marked `sensitive = true` in HCL get
  `/** @sensitive */` (TS), `# sensitive` (Python), or `// Sensitive`
  (Java/C#/Go) annotations in the generated code

#### Multi-source modules

For organizations with modules spread across multiple repositories, the
codegen supports a `--manifest` option:

```bash
npx tsx src/cli.ts generate \
  --manifest modules-manifest.json \
  --output-dir ../constructs/cdktf
```

The manifest file maps module IDs to repo URLs:

```json
{
  "modules": {
    "networking": { "repo": "git@github.com:nautilus/terraform-modules.git", "ref": "v1.5.0", "path": "modules" },
    "database":   { "repo": "git@github.com:nautilus/data-modules.git", "ref": "v2.0.0", "path": "modules/database" }
  }
}
```

The codegen clones each unique repo at the specified ref and discovers
modules within the given path. `--modules-dir` remains the default for
single-repo setups.

### 10. Write a README

Every module needs:
- One-paragraph description
- Usage example (CDKTF construct, not raw Terraform)
- Inputs table
- Outputs table

### 11. Open a PR

CI runs `fmt -check`, `validate`, and `terraform test` on every module automatically.
All checks must pass before merge.

---

## Modifying an existing module

### Backwards-compatible changes (minor or patch version)

Safe to release without a migration guide:
- Adding a new **optional** variable (with a default that preserves current behaviour)
- Adding a new output
- Adding a new resource that does not replace an existing one
- Fixing a bug that makes the module behave as documented

### Breaking changes (major version)

Require a major version bump, a written migration guide, and a two-week notice to
all consuming teams before release:
- Removing or renaming a variable
- Changing the type of a variable
- Removing or renaming an output
- Changing a resource's name or type (causes destroy + recreate)
- Adding a required variable (no default)
- Changing a `lifecycle.ignore_changes` list in a way that causes unexpected updates

When in doubt, treat it as breaking.

### How to make a breaking change safely

1. Add the new variable or output alongside the old one (both present).
2. Deprecate the old one in the README.
3. Release as a minor version with both present.
4. Give consuming teams one full sprint to upgrade.
5. Remove the old one in the next major version.

---

## Testing

### CI — always required

The CI workflow (`tf-modules/.github/workflows/ci.yml`) runs on every PR:
- `terraform fmt -check -recursive modules/`
- `terraform init -backend=false` + `terraform validate` per module
- `terraform test` per module

Run these locally before pushing:

```bash
terraform fmt -recursive modules/
cd modules/networking && terraform init -backend=false && terraform validate
cd modules/networking && terraform test
```

### Construct unit tests

Each language library synthesizes constructs to JSON and asserts the module call
is wired correctly. Run them when changing module variable or output names.

```bash
cd constructs/python && pip install -e ".[dev]" && pytest tests/ -v
cd constructs/typescript && npm install && npm test
cd constructs/csharp && dotnet test Nautilus.Infra.Tests/
cd constructs/java && mvn test --batch-mode
cd constructs/go && go test ./...
```

These tests catch mismatches between the construct (e.g. `subnet_id`) and the
module variable name (e.g. `delegated_subnet_id`).

### OPA policy tests

Run the OPA test suite before merging any policy change:

```bash
opa test policy/ -v
```

---

## Versioning and release process

The module repo uses Git tags following **Semantic Versioning** (`vMAJOR.MINOR.PATCH`).
Construct library package versions always match the module tag they reference.

### Release checklist

- [ ] All CI checks pass on `main`
- [ ] `CHANGELOG.md` updated under the new version heading
- [ ] Module `README.md` updated if any inputs or outputs changed
- [ ] `module.json` `moduleRef` updated to the new tag
- [ ] For breaking changes: migration guide written and distributed to all consuming teams
- [ ] For breaking changes: two-week notice period elapsed
- [ ] Tag pushed → codegen PR opened automatically as `nautilus[bot]`
- [ ] Codegen PR reviewed, CI passes, merged
- [ ] All five packages published to their respective internal registries (automated — see below)

### Tagging a release

```bash
git checkout main && git pull
git tag -s v1.5.0 -m "Release v1.5.0"
git push origin v1.5.0
```

Use signed tags (`-s`). The CI `tag-check` job validates the semver format.

### Updating constructs (automated)

When you push a new tag, the `terraform-modules` repo dispatches a
`module-release` event to `project-nautilus`. The codegen workflow:

1. Clones the modules repo at the new tag (configurable via the `MODULES_REPO`
   repository variable; defaults to `nautilus/terraform-modules`)
2. Reads `variables.tf`, `outputs.tf`, and `module.json` for each module
3. Generates updated construct code for all five languages
4. Opens a PR as `nautilus[bot]` with the changes

If a `--manifest` input is provided to the workflow dispatch, the codegen uses
multi-source mode instead of `--modules-dir`.

The `moduleRef` in each `module.json` must be updated to the new tag before
the codegen runs. Include this in the release commit:

```bash
# Update all module.json files to the new ref
find modules -name module.json -exec sed -i 's/"moduleRef": "v.*"/"moduleRef": "v1.5.0"/' {} +
git add -A && git commit -m "bump module refs to v1.5.0"
git tag -s v1.5.0 -m "Release v1.5.0"
git push origin main v1.5.0
```

The codegen PR triggers CI automatically (because it's opened by the Nautilus
App, not by `GITHUB_TOKEN`). Review, merge, and publish.

### Automated registry publishing

Each construct library's CI workflow (`constructs/<lang>/.github/workflows/ci.yml`)
publishes automatically when a semver tag is pushed. Publishing is gated on the
`test` job passing and on the `registry` GitHub Environment, which should be
configured with the platform team as required reviewers.

The workflow verifies the git tag matches the version declared in the package
manifest before publishing. A mismatch fails the job before any artifact is
uploaded.

| Language | Registry | Publish mechanism | Secret |
|----------|---------|-------------------|--------|
| Python | Internal PyPI (`pkgs.nautilus.internal`) | `twine upload` with `__token__` auth | `REGISTRY_TOKEN` |
| TypeScript | Internal npm (`npm.nautilus.internal`) | `npm publish` via `NODE_AUTH_TOKEN` | `REGISTRY_TOKEN` |
| Java | Internal Maven (`maven.nautilus.internal`) | `mvn deploy` with server credentials | `REGISTRY_TOKEN` |
| C# | Internal NuGet (`nuget.nautilus.internal`) | `dotnet nuget push` with API key | `REGISTRY_TOKEN` |
| Go | Internal Go proxy (`goproxy.nautilus.internal`) | Proxy cache warmed via HTTP request | `REGISTRY_TOKEN` |

Go is different from the other four: Go modules are versioned by git tag, not a
package manifest. The publish job warms the internal Athens proxy cache by
requesting the new version immediately after the tag is pushed. Consumers get the
module on their first `go get` without waiting for the proxy to fetch it lazily.

### Setting up the `registry` GitHub Environment

Each construct library repo needs a `registry` Environment configured before
publishing will work:

1. Repo **Settings → Environments → New environment** → name it `registry`
2. Add **Required reviewers**: platform team lead (or the full platform team)
3. Add the `REGISTRY_TOKEN` secret scoped to this environment

Scoping `REGISTRY_TOKEN` to the environment (rather than at the repo level)
ensures it is only accessible to the publish job and only after a reviewer approves.

---

## Access control

The `nautilus/terraform-modules` repo is private.

| GitHub team | Access |
|-------------|--------|
| `platform-infra` | Write (merge PRs, create tags) |
| Product team developers | No direct access |
| CI runners (product repos) | Read-only via per-repo SSH deploy key |

Each product repo has its own deploy key — keys are not shared across repos.

### Provisioning a deploy key for a new product team

```bash
# Generate a key pair (no passphrase)
ssh-keygen -t ed25519 -C "github-actions@nautilus/<product>-infra" -f /tmp/tf_modules_key

# Add public key as a read-only deploy key on the terraform-modules repo:
#   Settings → Deploy keys → Add deploy key → Allow write access: NO

# Add private key as a secret in the product team's repo:
#   Settings → Secrets and variables → Actions → New secret → TF_MODULES_DEPLOY_KEY
```

Rotate deploy keys annually and immediately on suspected compromise.

---

## Releasing reusable workflows

The `nautilus/reusable-workflows` repo is versioned independently from the Terraform
modules. Calling workflows (`infra.yml`) pin a specific tag:

```yaml
uses: nautilus/reusable-workflows/.github/workflows/tf-plan.yml@v1.0.0
```

### When to cut a new release

Cut a new reusable-workflows release when:
- A new workflow step is added or an existing step changes behaviour
- An action version is bumped (e.g. `actions/checkout@v4` → `v5`)
- A new workflow is added (e.g. `tf-drift.yml`)
- A bug in the pipeline logic is fixed

### Breaking vs. non-breaking changes

A change is **breaking** if it removes or renames a workflow input or secret that
calling workflows reference. Inputs added with defaults and entirely new workflows
are non-breaking. When in doubt, treat it as breaking.

Breaking changes require:
- A migration guide showing how to update `infra.yml` in each product repo
- Two-week notice to all product teams
- A coordinated PR push to every consuming repo (see the pipeline template update
  script in [Product Team Maintenance — Routine maintenance](platform-product-maintenance.md#routine-maintenance))

### Release checklist

- [ ] All CI checks pass on `main` (workflow trigger tests)
- [ ] Calling workflow in `tf-azure/` updated and tested end-to-end
- [ ] For breaking changes: migration guide written and distributed
- [ ] For breaking changes: two-week notice elapsed

### Tagging a release

```bash
git checkout main && git pull
git tag -s v1.1.0 -m "Release v1.1.0"
git push origin v1.1.0
```

The CI `tag-check` job validates the semver format on every tag push.

### Updating the pin in product repos

After tagging, open a PR on every product team's infra repo to update the
`@v1.x.x` pin across all `uses:` lines in their `infra.yml`. Use the bulk PR
script documented in [Product Team Maintenance — Routine maintenance](platform-product-maintenance.md#routine-maintenance).

---

## Maintaining enforcement policies

### OPA/conftest policies (`policy/`)

OPA policies are reviewed like any code change — a PR is required. The OPA test
suite (`*_test.rego`) must be updated alongside the policy and must pass before
merging:

```bash
opa test policy/ -v
```

**When to update OPA policies:**
- A new resource type must be blocked (add to the relevant type set)
- A new SKU is approved (add to `deny_budget_violations.rego`)
- An existing policy is too broad and blocking legitimate use

**Environment aliases in OPA:**

The policy library (`policy/lib/policy_enabled.rego`) automatically resolves
environment aliases before checking whether a policy is active. For example,
if `data.environment` is `"production"`, the library resolves it to `"prod"`
before comparing against the policy's environment list. Default aliases
(`production` -> `prod`, `uat` -> `staging`, `test` -> `dev`, `development`
-> `dev`) are built in and can be extended via `data.environment_aliases` in
`policy_defaults.json` or `nautilus.yaml`.

**Testing a policy change locally:**

```bash
# Get a real plan JSON to test against
terraform show -json tfplan > tfplan.json

# Run all policies against it
conftest test tfplan.json --policy policy/ --data <(printf '{"environment":"prod"}')
```

Always test both the case that should be denied and the case that should pass.

### Azure Policy (`tf-modules/governance/`)

Azure Policy is the ARM-level enforcement backstop. Changes to Azure Policy
definitions are Terraform changes in `tf-modules/governance/` and go through the
normal module PR and tag process.

The governance module is applied by the platform team from a privileged workspace
— not by product team pipelines:

```bash
cd tf-modules/governance
terraform init
terraform plan    # review carefully — this affects all subscriptions
terraform apply
```

**When to update Azure Policy:**
- A new resource type must be blocked at the ARM level
- Scope needs to change (new subscription added for a new environment)
- Budget caps require adjustment (`variables.tf → monthly_budget_amounts`)

### Management locks

Management locks are defined inside each Terraform module with a count gated on
`var.environment != "dev"`. They are created automatically when a product team's
stack applies in staging or prod. No separate management is required.

To remove a lock for a supervised decommission, see
[Product Team Maintenance — Supervised decommission](platform-product-maintenance.md#supervised-decommission-in-stagingprod).
