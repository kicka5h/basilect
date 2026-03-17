# Nautilus GitHub App Setup

The Nautilus GitHub App is the single identity behind all platform automation.
Every automated commit, PR, and API call in the Nautilus system uses this App.

---

## What the App does

| Feature | How the App is used |
|---------|-------------------|
| **Codegen PRs** | When a Terraform module is tagged, the codegen workflow generates updated constructs and opens a PR as `nautilus[bot]` |
| **Dashboard API** | The dashboard fetches repo, workflow, and PR data using the App's installation token (higher rate limits than PATs) |
| **Wiki sync** | Wiki updates are committed and pushed as `nautilus[bot]` |
| **CI triggering** | PRs opened by the App trigger CI checks (unlike `GITHUB_TOKEN`-opened PRs) |

---

## Creating the App

### Option A — automated (recommended)

```bash
bash setup/scripts/create-github-app.sh <your-org>
```

The script creates the App, saves the private key, and prints the secrets to
set. Requires the `gh` CLI authenticated as an org owner.

### Option B — manual

1. Go to **https://github.com/organizations/\<org\>/settings/apps/new**
2. Configure:

| Setting | Value |
|---------|-------|
| App name | `Nautilus` |
| Homepage URL | `https://github.com/<org>/project-nautilus` |
| Webhook | **Inactive** (uncheck "Active") |
| Where can this be installed? | Only on this account |

3. Set repository permissions:

| Permission | Access |
|-----------|--------|
| Administration | Read & Write |
| Contents | Read & Write |
| Metadata | Read |
| Pull requests | Read & Write |
| Issues | Read |
| Actions | Read |
| Checks | Read |
| Workflows | Read & Write |

4. Click **Create GitHub App**
5. Note the **App ID** from the settings page
6. Click **Generate a private key** — download the `.pem` file
7. Click **Install App** → select your organization → choose repositories

---

## Installation

After creating the App, install it on the organization and grant access to:

- `project-nautilus` (codegen, wiki sync)
- `terraform-modules` (codegen reads module source)
- All product team repos the dashboard should monitor

Note the **Installation ID** from the URL after clicking "Configure" on the
installed App (the number at the end of the URL).

---

## Setting secrets

Base64-encode the private key (avoids multiline secret issues):

```bash
base64 < nautilus-app.pem | tr -d '\n'
```

Set secrets on the `project-nautilus` repository:

```bash
# Codegen workflow + wiki sync
gh secret set NAUTILUS_APP_ID --body '<app-id>'
gh secret set NAUTILUS_APP_PRIVATE_KEY --body '<base64-encoded-pem>'

# Dashboard
gh secret set GH_APP_ID --body '<app-id>'
gh secret set GH_APP_PRIVATE_KEY --body '<base64-encoded-pem>'
gh secret set GH_APP_INSTALLATION_ID --body '<installation-id>'
```

The App ID and private key are the same for both the codegen and dashboard —
they just use different secret names for clarity.

---

## Verifying the App works

Test authentication by generating a JWT and calling the GitHub API:

```bash
# Generate a JWT (requires the gh CLI and jq)
APP_ID=<your-app-id>
PEM_FILE=<path-to-pem>

# The App should appear in the API response
gh api /app --jq '.name' \
  -H "Authorization: Bearer $(ruby -e "
    require 'openssl'; require 'jwt'
    key = OpenSSL::PKey::RSA.new(File.read('$PEM_FILE'))
    puts JWT.encode({iat: Time.now.to_i - 60, exp: Time.now.to_i + 600, iss: $APP_ID}, key, 'RS256')
  ")"
```

If you don't have Ruby/JWT available, the easiest check is to trigger the
codegen workflow manually and verify the PR appears as `nautilus[bot]`.

---

## Rotating the private key

1. Go to the App settings page
2. Click **Generate a private key** (this creates a new key without revoking the old one)
3. Update the `NAUTILUS_APP_PRIVATE_KEY` and `GH_APP_PRIVATE_KEY` secrets
4. Verify the codegen and dashboard still work
5. Revoke the old key from the App settings page

---

## Security considerations

- The private key is the only credential — protect it like a production secret
- The App has **read-only** access to contents and **write** access only to PRs and workflows
- Installation scope is limited to selected repositories, not the entire org
- All actions by the App appear in the GitHub audit log under `nautilus[bot]`
- Rotate the private key annually or immediately on suspected compromise
