# Product Team Maintenance

How the platform team supports product teams that run on Nautilus: reviewing
their PRs, managing state, handling incidents, onboarding new teams, and
coordinating module upgrades.

---

## What the platform team owns per product team

| Responsibility | Details |
|----------------|---------|
| CI/CD pipeline template | `infra.yml` — the team copies it, the platform team maintains it |
| Remote state backend | Blob Storage account, container, and RBAC |
| Azure provider credentials | Service principal per product, OIDC federated credentials |
| Module access | Deploy key provisioning (`TF_MODULES_DEPLOY_KEY`) |
| Production approval gate | Required reviewer on the `stage` and `prod` GitHub Environments |
| OPA/conftest policies | `policy/` rules that gate every `terraform apply` |
| Azure Policy governance | Policy definitions and assignments across all subscriptions |
| Management lock removal | Supervised decommissions of protected resources |
| Incident response | Failed applies, state corruption, unexpected drift |
| Module upgrades | Coordinating breaking changes across all consuming repos |

The product team owns their stack file, their package version pin, and their
application-level configuration.

---

## Reviewing a product team's PR

The platform team is a required reviewer for all PRs that touch infrastructure
in staging or prod environments. When reviewing:

### Structural checks

- [ ] Stack inherits from `BaseAzureStack`, not `TerraformStack` directly.
- [ ] `project` and `environment` are passed correctly to `super().__init__()`.
- [ ] No hardcoded secrets — passwords and tokens must come from `os.environ`.
- [ ] No raw `TerraformModule` or `TerraformResource` calls — only approved constructs.
- [ ] Stack outputs are descriptive and do not expose sensitive values as plaintext.

### Plan review

The pipeline posts a Terraform plan as a PR comment. Read it carefully:

```
# azurerm_kubernetes_cluster.this will be created
  + resource ...

# azurerm_postgresql_flexible_server.this must be replaced
-/+ resource ...   ← DESTRUCTIVE — investigate before approving
```

Red flags:

| Signal | Action |
|--------|--------|
| `-/+` (destroy and recreate) on a stateful resource (DB, cluster) | Block and discuss with the team |
| Unexpected new resources not mentioned in the PR description | Ask the developer to explain |
| Changes to `administrator_password` on an existing server | Confirm this is intentional rotation |
| `project` or `environment` values changed | Block — these are in resource names; changing them destroys everything |
| `?ref=` tag in module source doesn't match the declared construct library version | Block — version mismatch means undefined behaviour |

The OPA policy gate runs before this plan is posted. By the time you see the
plan, it has already passed automated policy checks. Your review focuses on
intent and correctness.

### Approving

If the plan looks correct, approve in GitHub. The `stage` and `prod` Environment
gates require a separate manual approval before apply runs.

---

## Handling policy violations

### A PR is blocked by the OPA policy gate

The pipeline annotates the specific violation on the PR diff. Common patterns:

- **`DENY [deletions-outside-dev]`**: If the deletion is intentional (decommission),
  follow the [supervised decommission](#supervised-decommission-in-stagingprod) process.
- **`DENY [network-outside-dev]`** or **`DENY [permissions-outside-dev]`**: The team
  needs to remove the offending resource from their stack. If there is a legitimate
  need, the platform team handles the change through a privileged pipeline.
- **`DENY [budget]`**: The team needs a different SKU, or the SKU needs to be added
  to `policy/deny_budget_violations.rego`.
- **`DENY [required-tags]`**: A resource type is missing required tags. If the type
  should be exempt (e.g. a new Azure resource type), add it to the exempt list in
  `policy/deny_missing_required_tags.rego`.

### An apply fails with an Azure Policy denial

This means the OPA gate was bypassed (e.g. someone ran `terraform apply` manually
outside the pipeline) or there is a gap in the OPA policies.

1. Identify the resource type and property from the error message.
2. If bypassed: investigate how and remediate access.
3. If an OPA gap: add the missing rule to the relevant `.rego` file and open a PR.

---

## Supervised decommission in staging/prod

Deleting resources in staging or prod requires three steps. Management locks
protect all non-dev resources from accidental destruction.

**Step 1 — Remove the management lock** (platform team only):

```bash
# List locks on the resource group
az lock list --resource-group <rg-name> --output table

# Remove the specific lock
az lock delete --name <lock-name> --resource-group <rg-name>
```

**Step 2 — Remove the resource from the stack and merge**:

The product team removes the resource from their stack file. The PR will now pass
the deletion policy because the lock removal is a deliberate, documented act.
Approve and apply.

**Step 3 — Confirm**:

After apply succeeds, confirm the resource is gone. The management lock does not
need to be re-created — the resource no longer exists.

Document the decommission (what was removed, why, who approved) in whatever
incident or change tracking system your team uses.

---

## Understanding the synthesized output

When debugging a pipeline failure, inspect the synthesized Terraform JSON.
Download the `cdktf-out-<sha>` artifact from the failed workflow run.

```
cdktf.out/
└── stacks/
    └── portal-stack/
        ├── cdk.tf.json          Main Terraform configuration
        └── .terraform.lock.hcl  Provider lock file (generated by init)
```

Inspect module calls, variable values, and backend configuration:

```bash
cat cdktf.out/stacks/portal-stack/cdk.tf.json | jq '.module'
```

To reproduce locally (requires Azure credentials and SSH key for modules):

```bash
cd cdktf.out/stacks/portal-stack
eval $(ssh-agent) && ssh-add /path/to/tf_modules_deploy_key
terraform init
terraform plan
```

---

## Managing remote state

State files live in the platform-managed Blob Storage account:

```
Storage account: platformtfstate
Container:       tfstate
  portal/dev/terraform.tfstate
  portal/staging/terraform.tfstate
  portal/prod/terraform.tfstate
  myapp/dev/terraform.tfstate
  ...
```

The state key (`{project}/{environment}/terraform.tfstate`) is set by
`BaseAzureStack` and cannot be changed by product teams.

### Viewing state

```bash
# Requires Storage Blob Data Contributor on the platformtfstate account
az storage blob download \
  --account-name platformtfstate \
  --container-name tfstate \
  --name portal/prod/terraform.tfstate \
  --file portal-prod.tfstate

terraform show -json portal-prod.tfstate | jq '.values.root_module'
```

### State lock

Terraform uses blob lease-based locking. If a pipeline job is cancelled mid-run,
the lease may persist. Before breaking a lease, confirm with the product team
that no pipeline is actually running.

```bash
# Check the lease state on the blob
az storage blob show \
  --account-name platformtfstate \
  --container-name tfstate \
  --name portal/prod/terraform.tfstate \
  --query "properties.lease"

# Break the lease (requires Storage Account Contributor)
az storage blob lease break \
  --account-name platformtfstate \
  --container-name tfstate \
  --name portal/prod/terraform.tfstate
```

### State surgery

If state becomes inconsistent — for example, a resource was deleted in Azure
manually but still exists in state — use `terraform state rm` to remove the stale
reference, then re-import or let the next apply recreate it.

```bash
# Remove a stale resource from state
terraform state rm 'module.networking.azurerm_subnet.this["db"]'

# Import an existing resource into state
terraform import \
  'module.networking.azurerm_subnet.this["db"]' \
  /subscriptions/<id>/resourceGroups/portal-prod-rg/providers/Microsoft.Network/...
```

State surgery must always be performed with the product team informed and on
standby. Keep a written record of every change made and why.

---

## Handling a failed apply

### 1. Assess the failure

Pull up the failed GitHub Actions run. The `terraform apply` step output shows
which resource failed and why.

| Failure | Likely cause |
|---------|-------------|
| `AuthorizationFailed` | Service principal lacks a required RBAC role |
| `ResourceGroupNotFound` | Networking module has not created the RG yet (ordering issue) |
| `QuotaExceeded` | Subscription vCPU quota insufficient for the requested VM size |
| `InvalidParameter` | A variable value violates an Azure constraint (e.g. password complexity) |
| `ResourceAlreadyExists` | A resource with the same name was created outside Terraform |

### 2. Determine what was created

Terraform applies are not atomic. Some resources may have been created before the
failure. Check the state file to see what succeeded:

```bash
terraform show -json portal-prod.tfstate \
  | jq '[.values.root_module.child_modules[].resources[].address]'
```

### 3. Fix and re-run

Fix the root cause (RBAC assignment, quota request, corrected variable value).
Re-trigger the pipeline from the same commit — do not force-push or amend.

> **Important:** If the `tfplan` artifact has expired (default 3-day retention,
> 90 days for prod), re-run from the `synth` job to generate a fresh plan.
> Review the new plan before approving apply.

### 4. If partial apply left inconsistent state

Inform the product team. Do not attempt manual remediation without coordinating
with the full platform team. Follow the [state surgery](#state-surgery) procedure
if needed.

---

## Emergency (break-glass) deployment

> Use this only when the pipeline is broken and a production fix cannot wait.
> Every break-glass deployment must be logged and reviewed post-incident.
> Two platform team members must be present.

### When to use

- The GitHub Actions runner is offline and a critical production fix must be deployed.
- The reusable workflow has a bug that blocks all deploys and a hotfix is urgent.
- Azure Policy or a management lock is blocking a legitimate emergency change.

### Prerequisites

- Platform team lead authorization (captured in the incident record before touching anything).
- A second platform engineer present as witness.
- Your personal Azure credentials configured with Contributor scope on the target
  subscription. The OIDC service principal used by the pipeline does not work
  interactively — use `az login` with your personal identity.

### Steps

**1. Open an incident record** before touching anything. Capture: reason for
break-glass, environment affected, change being deployed, names of both engineers.

**2. Remove the management locks** (non-dev only). Locks exist on the resource
group, VNet, PostgreSQL server, and AKS cluster. Remove only those needed for
this specific change — leave all others in place.

```bash
az lock list --resource-group <rg-name> --output table
az lock delete --name <lock-name> --resource-group <rg-name>
```

**3. Authenticate and initialize**:

```bash
az login
az account set --subscription <subscription-id>

eval $(ssh-agent) && ssh-add /path/to/tf_modules_deploy_key

terraform -chdir=shared init -backend-config=../<env>/backend.hcl
```

**4. Plan and review**:

```bash
terraform -chdir=shared plan \
  -var-file=../<env>/terraform.tfvars \
  -out=emergency.tfplan
```

Confirm the plan matches exactly what the incident record describes. If it shows
anything unexpected, stop and investigate before applying.

**5. Apply**:

```bash
terraform -chdir=shared apply emergency.tfplan
```

**6. Re-create every management lock you removed**:

```bash
az lock create \
  --name "<prefix>-rg-lock" \
  --resource-group <rg-name> \
  --lock-type CanNotDelete \
  --notes "Restored after break-glass: <INCIDENT-REF>"

az lock create \
  --name "<prefix>-vnet-lock" \
  --resource-id <vnet-resource-id> \
  --lock-type CanNotDelete \
  --notes "Restored after break-glass: <INCIDENT-REF>"
# Repeat for PostgreSQL server and AKS cluster as needed
```

**7. Post-incident** (within 24 hours):
- Close the incident record with the apply output and a summary.
- Open a follow-up to fix the underlying pipeline issue.
- Run the drift detection workflow manually to confirm state is consistent.
- Present the incident at the next platform team meeting to prevent recurrence.

---

## Handling unexpected drift

Drift occurs when the real Azure state diverges from Terraform state — typically
because someone created, modified, or deleted a resource outside the pipeline.

The daily drift detection workflow (`tf-drift.yml`) runs `terraform plan
-detailed-exitcode` against every environment. If changes are detected (exit code 2),
it opens a GitHub issue labelled `drift` and the environment name, or adds a comment
to an existing open drift issue.

### Resolving drift

1. **Identify the drift**: read the plan output in the GitHub issue.
2. **Decide the source of truth**:
   - If Terraform should win: merge with no stack changes. The apply will overwrite
     the manual change.
   - If the manual change should win: update the stack code to match reality,
     then plan and apply.
3. **Close or update the drift issue** once the state is reconciled.

Never leave drift unresolved across multiple apply cycles. It compounds.

---

## Onboarding a new product team

### Platform team tasks

- [ ] Create a service principal for the team's Azure subscription with the minimum
  required roles: Contributor on the target resource group scope, plus Storage Blob
  Data Contributor on the state account.
- [ ] Configure OIDC federated credentials on the SP for each environment:
  subject `repo:<org>/<product>-infra:environment:<env>`.
- [ ] Add the SP details as GitHub Actions secrets on the team's repo:
  `ARM_CLIENT_ID`, `ARM_SUBSCRIPTION_ID` (per env prefix), and the shared
  `ARM_TENANT_ID` repo variable.
- [ ] Generate a read-only SSH deploy key for `nautilus/terraform-modules`. Add the
  public key as a deploy key on the module repo. Add the private key as
  `TF_MODULES_DEPLOY_KEY` in the team's repo.
- [ ] Pre-create the Blob Storage state container path:
  `tfstate/<project>/<environment>/terraform.tfstate`
- [ ] Configure the `stage` and `prod` GitHub Environments on the team's repo with
  the platform team as required reviewer and a 10-minute review window.
- [ ] Share `LOG_WORKSPACE_ID` for the platform-managed Log Analytics workspace.

### Product team tasks (guided by platform team)

- [ ] Set up the repo structure: `cdktf.json`, dependency file, `stacks/`.
- [ ] Copy `infra.yml` from the reference implementation and update stack name references.
- [ ] Write a minimal stack (network only), synthesize locally, open a PR.
- [ ] Platform team reviews and approves the first PR end-to-end together.

The first PR should be a dev-environment, network-only stack to validate the full
pipeline before adding database or compute resources.

---

## Coordinating module upgrades across teams

When releasing a `nautilus-infra` version that contains breaking changes:

1. **Announce** to all consuming teams at least two weeks before the release,
   with the migration guide attached.
2. **Publish** the new package versions to all internal registries.
3. **Notify each team** with the specific changes they need to make to their stack file.
4. **Support teams** during their upgrade PRs — be available for plan review.
5. **Set a hard deadline** (typically end of the sprint after announcement). After
   that, the old module tag may be archived.
6. **Verify** all teams have merged their upgrades before archiving the old tag.

Track each team's upgrade status (pending / PR open / merged) in whatever tracking
system your team uses until all teams are on the new version.

---

## Routine maintenance

### Monthly

- [ ] Review state file sizes. Very large states (> 10 MB) may indicate resource
  sprawl — review with the relevant product team.
- [ ] Audit deploy key access — confirm no stale keys exist for decommissioned repos.
- [ ] Check for Terraform provider updates and plan a minor version bump if needed.

### Quarterly

- [ ] Review Kubernetes versions in use and identify clusters approaching end-of-support.
  Notify teams with a planned upgrade window.
- [ ] Review PostgreSQL versions in use. Azure ends support on a rolling schedule.
- [ ] Run `terraform validate` on all modules against the latest provider version.
- [ ] Rotate deploy keys.

### Ad hoc — pipeline template updates

When updating `infra.yml` (new action version, new CI step), push the update to
every consuming repo. Use a script to open identical PRs:

```bash
for repo in portal-infra myapp-infra billing-infra; do
  gh pr create \
    --repo nautilus/$repo \
    --title "chore: update infra pipeline to v2.1" \
    --body "Platform team maintenance — no stack changes required." \
    --base main \
    --head platform/pipeline-v2.1
done
```
