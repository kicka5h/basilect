# Nautilus Dashboard

The Nautilus Dashboard is a self-hosted Next.js application that gives platform and product teams visibility into the state of their infrastructure repos.

---

## Pages

| Page | Route | What it shows |
|------|-------|---------------|
| **Repository overview** | `/` | Card grid of all `nautilus-managed` repos with last deploy status, drift alerts, open PRs |
| **Environment health** | `/repos/[owner]/[repo]` | Per-environment (dev/qa/staging/prod) deploy status, drift status, recent workflow runs |
| **Policy compliance** | `/compliance` | Aggregated OPA violations across repos, weekly trend chart, filter by repo |
| **Module versions** | `/versions` | Matrix of repos × construct version, outdated flags, adoption percentage |
| **Open PRs** | `/pulls` | All open PRs across Nautilus-managed repos with CI status |
| **Activity feed** | `/activity` | Timeline of deploys, drift events, and codegen PRs across all repos |
| **Policies** | `/policies` | Policy catalog and recent violations |
| **Settings** | `/settings` | User management — add/remove users, change roles (admin only) |

---

## First-time setup

### Prerequisites

- Node.js 18+
- A GitHub App configured for Nautilus (see [GitHub App Setup](github-app-setup.md))
- The App's **App ID**, **private key** (`.pem` file), and **Installation ID**

### 1. Install dependencies

```bash
cd dashboard
npm install
```

### 2. Start the dev server

```bash
npm run dev   # http://localhost:3000
```

On first launch, the dashboard detects that no GitHub App is configured and
redirects to the **setup wizard** at `/setup`.

### 3. Complete the setup wizard

The wizard walks through five steps:

| Step | What happens |
|------|-------------|
| **Sign in to GitHub** | Opens a GitHub login popup so you can create/install the App under the right account |
| **Create GitHub App** | Shows the required permissions and opens GitHub's App creation page (or skip if you already have one) |
| **Enter credentials** | Paste the App ID, private key (.pem or base64), app slug, and optionally the Installation ID. You also name your **organization** — this is how the org appears in the dashboard |
| **Install the App** | Install the App on your GitHub org and enter the Installation ID from the URL |
| **Connect repositories** | Toggle which repos the dashboard should monitor (adds the `nautilus-managed` topic) |

After completing setup:
- A **Default** organization is created in the local SQLite database
- The GitHub App credentials are stored in the org record
- If authentication providers are configured (see below), the first user to
  sign in is automatically promoted to **admin**

### 4. (Optional) Configure authentication

Without auth providers, the dashboard is open to anyone who can reach it.
For private deployments, configure one or both:

**Azure AD:**
```env
AZURE_AD_CLIENT_ID=<client-id>
AZURE_AD_CLIENT_SECRET=<client-secret>
AZURE_AD_TENANT_ID=<tenant-id>
```

**GitHub OAuth:**
```env
GITHUB_CLIENT_ID=<client-id>
GITHUB_CLIENT_SECRET=<client-secret>
```

**NextAuth secret (required for any auth provider):**
```env
NEXTAUTH_SECRET=$(openssl rand -base64 32)
NEXTAUTH_URL=http://localhost:3000
```

Once auth is enabled, only users added to an organization (via Settings) can access the dashboard. The first user to sign in after setup becomes admin.

---

## Multi-organization support

The dashboard supports multiple organizations, each with its own GitHub App installation. This is useful when a single dashboard instance serves several GitHub orgs or teams with separate repos.

### Creating additional organizations

1. Go to **Settings** → the admin user management page
2. Or call the API directly:

```bash
curl -X POST http://localhost:3000/api/orgs \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Team Alpha",
    "appId": "123456",
    "privateKey": "<base64-encoded-pem>",
    "installationId": "78901234"
  }'
```

### Switching organizations

If a user belongs to multiple organizations, an **org switcher** dropdown appears in the nav bar next to the Nautilus logo. Click the org name to switch.
All dashboard data (repos, compliance, versions, etc.) is scoped to the currently selected organization.

---

## User management

User access is managed per organization. Only admins can add or remove users.

### Via the Settings page

1. Click **Settings** in the top nav bar (visible to admins)
2. Enter an email address and select a role (admin or member)
3. Click **Add**

Users can be removed or have their role changed from the same page.

### Via the API

| Action | Method | Endpoint |
|--------|--------|----------|
| List org users | GET | `/api/orgs/{orgId}/users` |
| Add user | POST | `/api/orgs/{orgId}/users` with `{ "email": "...", "role": "admin|member" }` |
| Change role | PATCH | `/api/orgs/{orgId}/users/{email}` with `{ "role": "admin|member" }` |
| Remove user | DELETE | `/api/orgs/{orgId}/users/{email}` |

**Safeguards:**
- The last admin of an organization cannot be demoted or removed
- Users not in any organization are rejected at sign-in with a
  "Contact your admin" message

### Roles

| Role | Can view dashboard | Can manage users | Can create orgs |
|------|--------------------|------------------|-----------------|
| **admin** | Yes | Yes | Yes |
| **member** | Yes | No | No |

---

## How it discovers repos

Repos are discovered via the GitHub topic `nautilus-managed`. Add this topic to any repo you want the dashboard to monitor:

```bash
gh repo edit <owner>/<repo> --add-topic nautilus-managed
```

The dashboard paginates through all repos accessible to the GitHub App installation — there is no 100-repo cap. Archived repos are automatically filtered out since they show stale data and reject upgrade PRs.

### Forked construct detection

If a repo has no standard package manifest (package.json, pyproject.toml, etc.) but contains source files with `?ref=vX.Y.Z` module references, the dashboard identifies it as using a **forked** construct — inline module source strings copied directly into the codebase rather than installed via the construct library. These repos appear with a "Forked" badge on the compliance page.

---

## Architecture

```
Browser → Next.js (Azure Static Web Apps)
            │
            ├── NextAuth.js (Azure AD + GitHub OAuth)
            │
            ├── SQLite (better-sqlite3)
            │   ├── orgs: GitHub App configs per organization
            │   └── org_users: email + role memberships
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

- **Database** — SQLite via `better-sqlite3`, stored at `.nautilus/nautilus.db`.
  Holds organization configs and user memberships. Auto-created on first access
- **Legacy migration** — if the DB is empty on startup but env vars or
  `.nautilus/config.json` have a valid config, a "Default" org is auto-created
  with those credentials
- **Caching** — server-side TTL cache (keyed by `orgId:` prefix) +
  `Cache-Control` headers + client-side SWR
- **Auth** — NextAuth.js with Azure AD and/or GitHub OAuth. Optional — the
  dashboard works without auth providers (open access, first org used by default)
- **Data source** — Nautilus GitHub App (not PAT) for higher rate limits
- **Rate limiting** — `@octokit/plugin-throttling` automatically retries on
  429 responses (up to 2 retries on primary rate limits, 1 on secondary)
- **Credential validation** — GitHub App credentials are validated via
  `apps.getAuthenticated()` before being persisted

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
npm install
npm run dev   # http://localhost:3000
```

The setup wizard runs automatically on first access. You can also pre-configure via environment variables or `.nautilus/config.json`:

```json
{
  "appId": "123456",
  "privateKey": "<base64-encoded-pem>",
  "installationId": "78901234",
  "appSlug": "nautilus",
  "appUrl": "https://github.com/apps/nautilus"
}
```

---

## API routes

| Route | Data source | Cache TTL |
|-------|------------|-----------|
| `GET /api/repos` | Search repos + latest deploy + drift issues + open PRs | 5 min |
| `GET /api/repos/[o]/[r]/environments` | Workflow runs + jobs + drift issues | 2 min |
| `GET /api/repos/[o]/[r]/versions` | Contents API (manifest files) | 1 hour |
| `GET /api/repos/[o]/[r]/workflows` | Workflow runs + failure details | 2 min |
| `GET /api/compliance` | Repo versions vs latest version | 5 min |
| `GET /api/pulls` | Open PRs per repo | 1 min |
| `GET /api/versions` | Aggregated versions across all repos | 1 hour |
| `GET /api/activity` | Deploy events + drift issues + codegen PRs | 2 min |
| `GET /api/alerts` | Aggregated drift/deploy/version/policy alerts | 2 min |
| `GET /api/policies` | Failed PR runs with policy annotations | 5 min |
| `GET /api/policies/catalog` | Policy definitions from `policy/policies.json` | 1 hour |
| `GET /api/orgs` | User's organizations | — |
| `POST /api/orgs` | Create a new organization | — |
| `GET /api/orgs/[id]/users` | List org members | — |
| `POST /api/orgs/[id]/users` | Add user to org | — |
| `PATCH /api/orgs/[id]/users/[email]` | Change user role | — |
| `DELETE /api/orgs/[id]/users/[email]` | Remove user from org | — |
| `POST /api/auth/switch-org` | Switch active organization (sets cookie) | — |

All data routes require authentication when auth providers are configured.
Org/user management routes require admin role.

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

This creates a CNAME record and configures the SWA custom domain with automatic TLS.
