# Nautilus Test Environment

Provisions and tears down a full multi-cloud, multi-org test environment for Nautilus.

## What it creates

| Layer | Azure | GCP | Digital Ocean |
|-------|-------|-----|---------------|
| Network | VNet + subnets + NSGs | VPC + subnets + firewall | VPC |
| Compute | AKS (1-node B2s) | GKE Autopilot | DOKS (1-node s-2vcpu-2gb) |
| Database | PostgreSQL Flexible (B1ms) | Cloud SQL (f1-micro) | Managed PG (1vcpu-1gb) |
| State | Storage Account (LRS) | GCS bucket | Spaces bucket |

**GitHub** (2 orgs):
- `k1cka5h`: project-nautilus, terraform-modules, portal-infra, billing-infra
- `heyk1cka5h`: same repos, separate GitHub App

**Estimated daily cost**: ~$14/day total (Azure ~$8, GCP ~$5, DO ~$1)

## Quick start

```bash
# 1. Configure
cp testing/config.env testing/config.local.env
# Edit config.local.env with your subscription IDs, project IDs, tokens

# 2. Setup everything
bash testing/setup.sh

# Or setup individual components:
bash testing/setup.sh github          # just GitHub repos
bash testing/setup.sh azure           # just Azure infra
bash testing/setup.sh gcp             # just GCP infra
bash testing/setup.sh digitalocean    # just DO infra
bash testing/setup.sh dashboard       # just dashboard config
```

## Teardown

```bash
# Dry run (shows what would be destroyed)
bash testing/teardown.sh

# Actually destroy everything
bash testing/teardown.sh --confirm

# Destroy specific components
bash testing/teardown.sh azure --confirm
bash testing/teardown.sh gcp digitalocean --confirm
bash testing/teardown.sh github --confirm
```

## Prerequisites

| Tool | Required for | Install |
|------|-------------|---------|
| `terraform` >= 1.5 | All clouds | [terraform.io](https://terraform.io) |
| `gh` CLI | GitHub setup | `brew install gh` |
| `jq` | Everything | `brew install jq` |
| `az` CLI | Azure | `brew install azure-cli` |
| `gcloud` CLI | GCP | [cloud.google.com/sdk](https://cloud.google.com/sdk) |
| `doctl` CLI | DO (optional) | `brew install doctl` |

## GitHub multi-account auth

The setup requires switching between two GitHub accounts. Use `gh auth login`
before each org setup step — the script pauses and prompts you.

```bash
# For kicka5h / k1cka5h
gh auth login --hostname github.com

# For heykicka5h / heyk1cka5h
gh auth login --hostname github.com
```

## File structure

```
testing/
├── setup.sh                 # Main orchestration — provisions everything
├── teardown.sh              # Main teardown — destroys everything
├── config.env               # Template config (committed)
├── config.local.env         # Your config with secrets (gitignored)
├── .gitignore
├── github/
│   ├── setup-org.sh         # Create repos, apply config per org
│   └── teardown-org.sh      # Delete repos per org
├── azure/
│   ├── main.tf              # VNet + AKS + PostgreSQL + state (uses existing tf-modules)
│   ├── variables.tf
│   └── terraform.tfvars.example
├── gcp/
│   ├── main.tf              # VPC + GKE Autopilot + Cloud SQL + GCS
│   ├── variables.tf
│   └── terraform.tfvars.example
└── digitalocean/
    ├── main.tf              # VPC + DOKS + Managed PG + Spaces
    ├── variables.tf
    └── terraform.tfvars.example
```

## Notes

- All cloud resources use the cheapest available SKUs with no HA
- `force_destroy` / `deletion_protection = false` is set on all stateful resources
- Azure reuses existing `tf-modules/` for full parity with production module structure
- GCP uses Autopilot GKE (cheapest — pay per pod, not per node)
- DO Kubernetes uses a single `s-2vcpu-2gb` node ($12/mo)
- GitHub Apps must be deleted manually via org settings (API doesn't support it)
