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
    uses: 434marketing/.github/.github/workflows/wpe-deploy-staging.yml@v1
    with:
      theme_path: themes/my-theme/
      remote_path: wp-content/themes/my-theme/
    secrets: inherit
```

See `workflow-templates/` for the full starter files.

## Versioning

Site repos reference these workflows **by tag, never by branch**.

| Reference | Behavior | Use when |
|---|---|---|
| `@v1` | **Default.** Tracks the newest `v1.x.y`. Fixes and features arrive automatically; breaking changes never do. | Almost always |
| `@v1.2.3` | Frozen at one release. Nothing reaches the site until someone bumps it by hand. | A site is mid-migration, or you need a byte-reproducible deploy |
| `@main` | Tracks every commit as it lands. | Never — this is what tagging replaced |

### What counts as a breaking change

| Bump | Meaning |
|---|---|
| **MAJOR** (`v2`) | Renamed or removed inputs, a new required secret or variable, or any change that could break a site working today |
| **MINOR** (`v1.x`) | New optional inputs, new capability, or a changed default that is safe for every site |
| **PATCH** (`v1.0.x`) | Bug fixes with no interface change |

Sites on `@v1` inherit MINOR and PATCH **automatically**. That is the point of the floating
tag — but it means the bar for MAJOR is *"could this break a site that works today"*, not
*"does this feel like a big change"*. When in doubt, cut a major.

### Cutting a release

1. Merge to `main` (requires a PR and one approval).
2. Tag it and push:
   ```bash
   git tag v1.1.0 && git push origin v1.1.0
   ```
3. [`update-major-tag.yml`](.github/workflows/update-major-tag.yml) force-moves `v1` to that
   commit automatically. **This is the step that actually delivers the release** — every site
   on `@v1` picks it up on its next deploy.
4. Publish a GitHub Release with notes. Site owners read these to decide whether they need to
   pin away from `@v1`.

### Migrating to a new major

Breaking changes ship as `v2` and reach nobody until each site's `uses:` line is updated.
Keep shipping fixes to `v1` for a reasonable window so sites are not forced to migrate on a
deadline they did not choose.

### A note on trust

Every site passes `secrets: inherit`, so these workflows receive that site's
`WPE_SSHG_KEY_PRIVATE` and WP Engine API credentials. Anyone able to merge here — or to move
the `v1` tag — can reach the deploy credentials of every 434 site. That is why `main` requires
a PR and an approving review, and why those rules should not be relaxed for convenience.

For a site where even that is too much trust, pin to a full commit SHA. A tag can be moved;
a SHA cannot:

```yaml
uses: 434marketing/.github/.github/workflows/wpe-deploy-prod.yml@08d0020e907e0f3e849f0c48cc1a9df3a69dd307
```
