# 434 Marketing — Org-Wide GitHub Defaults

This repository contains shared GitHub Actions workflows and starter templates used across all [434 Marketing](https://434marketing.com) client site repositories.

## Reusable Workflows

These workflows are called from individual site repos and handle all deploy logic centrally.

### `wpe-deploy-staging.yml`

Deploys a WordPress theme to WP Engine staging. Triggered on pull request open/sync.

**Inputs:**

| Name | Description | Example |
|---|---|---|
| `theme_path` | Source path relative to repo root | `themes/my-theme/` |
| `remote_path` | Destination path on WP Engine | `wp-content/themes/my-theme/` |

**Required secrets:** `WPE_SSHG_KEY_PRIVATE`

**Required variables:** `WPE_STAGE_ENV`

---

### `wpe-deploy-prod.yml`

Backs up and deploys a WordPress theme to WP Engine production on PR merge. Creates a versioned GitHub release automatically.

**Inputs:** same as staging (`theme_path`, `remote_path`)

**Required secrets:** `WPE_SSHG_KEY_PRIVATE`, `WPE_API_USER_ID`, `WPE_API_PASSWORD`, `WPE_INSTALL_ID`

**Required variables:** `WPE_PROD_ENV`, `BACKUP_EMAIL_NOTIFICATION` (org-level)

**Optional secret:** `SLACK_WEBHOOK_URL` — if present, sends a Slack alert on failure

**Versioning:** reads PR labels (`major`, `minor`, `patch`) to determine the semver bump. Defaults to `patch`.

## Using These Workflows

In a site repo, the caller workflow is a thin file that passes site-specific paths:

```yaml
jobs:
  deploy:
    uses: 434marketing/.github/.github/workflows/wpe-deploy-staging.yml@main
    with:
      theme_path: themes/my-theme/
      remote_path: wp-content/themes/my-theme/
    secrets: inherit
```

See `workflow-templates/` for the full starter files.
