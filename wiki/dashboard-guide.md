# Nautilus Dashboard

The Nautilus Dashboard is a self-hosted Next.js application that gives platform
and product teams visibility into the state of their infrastructure repos.

---

## Pages

| Page | Route | What it shows |
|------|-------|---------------|
| **Repository overview** | `/` | Card grid of all `nautilus-managed` repos with last deploy status, drift alerts, open PRs |
| **Environment health** | `/repos/[owner]/[repo]` | Per-environment (dev/qa/staging/prod) deploy status, drift status, recent workflow runs |
| **Policy compliance** | `/compliance` | Aggregated OPA violations across repos, weekly trend chart, filter by repo |
| **Module versions** | `/versions` | Matrix of repos × construct version, outdated flags, adoption percentage |
| **Open PRs** | `/pulls` | All open PRs across Nautilus-managed repos with CI status |

---

## How it discovers repos

Repos are discovered via the GitHub topic `nautilus-managed`. Add this topic to
any repo you want the dashboard to monitor:

```bash
gh repo edit <owner>/<repo> --add-topic nautilus-managed
```

The dashboard paginates through all repos accessible to the GitHub App
installation — there is no 100-repo cap. Archived repos are automatically
filtered out since they show stale data and reject upgrade PRs.

### Forked construct detection

If a repo has no standard package manifest (package.json, pyproject.toml, etc.)
but contains source files with `?ref=vX.Y.Z` module references, the dashboard
identifies it as using a **forked** construct — inline module source strings
copied directly into the codebase rather than installed via the construct
library. These repos appear with a "Forked" badge on the compliance page.

---

## Architecture

```
Browser → Next.js (Azure Static Web Apps)
            │
            ├── NextAuth.js (Azure AD + GitHub OAuth)
            │
            └── API routes → GitHub API (via Nautilus GitHub App)
                              │
                              ├── Search repos by topic
                              ├── Workflow runs → deploy status
                              ├── Issues with "drift" label → drift alerts
                              ├── Failed PR runs → policy violations
                              ├── Package manifests → construct versions
                              └── Open PRs per repo
```

- **No database** — all data is fetched from GitHub on demand
- **Caching** — server-side TTL cache + `Cache-Control` headers + client-side SWR
- **Auth** — NextAuth.js with Azure AD and/or GitHub OAuth (configurable)
- **Data source** — Nautilus GitHub App (not PAT) for higher rate limits
- **Rate limiting** — `@octokit/plugin-throttling` automatically retries on
  429 responses (up to 2 retries on primary rate limits, 1 on secondary)
- **Credential validation** — GitHub App credentials are validated via
  `apps.getAuthenticated()` before being persisted, preventing silent failures
  on subsequent API calls

---

## Deployment

### 1. Deploy the infrastructure

```bash
cd dashboard/infra
terraform init
terraform apply \
  -var="resource_group_name=nautilus-dashboard-rg" \
  -var="github_app_id=<app-id>" \
  -var="github_app_private_key=<base64-pem>" \
  -var="github_app_installation_id=<installation-id>" \
  -var="nextauth_secret=$(openssl rand -base64 32)"
```

This creates an Azure Static Web App and optionally an Entra ID app
registration for Azure AD login.

### 2. Set repository secrets

```bash
gh secret set SWA_DEPLOYMENT_TOKEN --body "$(terraform output -raw swa_deployment_token)"
gh secret set NEXTAUTH_SECRET --body "$(openssl rand -base64 32)"
gh secret set GH_APP_ID --body '<app-id>'
gh secret set GH_APP_PRIVATE_KEY --body '<base64-pem>'
gh secret set GH_APP_INSTALLATION_ID --body '<installation-id>'
```

For Azure AD login, also set `AZURE_AD_CLIENT_ID`, `AZURE_AD_CLIENT_SECRET`,
and `AZURE_AD_TENANT_ID`. For GitHub OAuth, set `GH_OAUTH_CLIENT_ID` and
`GH_OAUTH_CLIENT_SECRET`.

### 3. Push to deploy

```bash
git push origin main  # triggers .github/workflows/deploy-dashboard.yml
```

### 4. Local development

```bash
cd dashboard
cp .env.example .env.local  # fill in your App credentials
npm install
npm run dev                  # http://localhost:3000
```

---

## API routes

| Route | Data source | Cache TTL |
|-------|------------|-----------|
| `GET /api/repos` | Search repos + latest deploy + drift issues + open PRs | 5 min |
| `GET /api/repos/[o]/[r]/environments` | Workflow runs + jobs + drift issues | 2 min |
| `GET /api/repos/[o]/[r]/versions` | Contents API (manifest files) | 1 hour |
| `GET /api/compliance` | Failed PR workflow runs across repos | 10 min |
| `GET /api/pulls` | Open PRs per repo | 1 min |
| `GET /api/versions` | Aggregated versions across all repos | 1 hour |

---

## Custom domain

To use a custom domain (e.g., `nautilus.example.com`):

```bash
terraform apply \
  -var="custom_domain=nautilus.example.com" \
  -var="dns_zone_name=example.com" \
  -var="dns_zone_resource_group=dns-rg" \
  # ... other vars
```

This creates a CNAME record and configures the SWA custom domain with
automatic TLS.
