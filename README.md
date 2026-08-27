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

**Versioning:** tags and releases **the calling site's repo**, reading that PR's labels
(`major`, `minor`, `patch`) to determine the semver bump. Defaults to `patch`. This is unrelated
to how *this* repo is versioned — see [Versioning](#versioning) below.

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

Releases are automatic. You never tag this repo by hand.

[`release-please.yml`](.github/workflows/release-please.yml) runs on every push to `main` and
keeps **one open Release PR** that accumulates the pending changes and shows the version they
will cut. Merging that PR is the act of releasing: it writes `CHANGELOG.md`, bumps
[`.github/VERSION`](.github/VERSION), creates the `vX.Y.Z` tag and the GitHub Release, and then
force-moves `v1` to it.

So the whole procedure is:

1. Merge a PR into `main` with a **conventional commit title** (see below).
2. Merge the Release PR when you want to ship. Everything else happens on its own.

`main` allows squash and rebase merges only, and squash is the normal path — which means the
**PR title becomes the commit message**, and that title is the only thing release-please reads.
[`pr-title-lint.yml`](.github/workflows/pr-title-lint.yml) rejects a PR whose title will not
parse, and its check summary tells you which bump the title is about to cause.

#### Writing the title

| Title | Bump | Reaches `@v1` sites |
|---|---|---|
| `feat!: require a WPE_INSTALL_ID secret` | MAJOR → `v2.0.0` | **Never** — each site must update its `uses:` line |
| `feat: add an optional php_lint input` | MINOR → `v1.2.0` | Automatically |
| `fix: correct the rsync exclude for lockfiles` | PATCH → `v1.1.1` | Automatically |
| `deps: bump actions/checkout from 6 to 7` | PATCH | Automatically |
| `docs:`, `perf:`, `refactor:`, `revert:` | PATCH | Automatically |
| `chore:`, `ci:`, `build:`, `test:`, `style:` | **None** | Never — the change sits on `main` |

A `!` before the colon, or a `BREAKING CHANGE:` footer, is what cuts a major. Use the test from
[What counts as a breaking change](#what-counts-as-a-breaking-change): *could this break a site
that works today* — not *does this feel big*.

#### Which types release, and why

release-please's rule is not "only `feat` and `fix` release." It is: **a type releases if that
type is visible in `changelog-sections`.** A window of commits whose types are all hidden
generates empty release notes, and release-please skips the release entirely
([`base.ts`](https://github.com/googleapis/release-please/blob/main/src/strategies/base.ts) —
`changelogEmpty`). Anything visible that is not `feat` and not breaking falls through to a
patch.

So the hidden list in [`.github/release-please-config.json`](.github/release-please-config.json)
*is* the release policy. It is set so that **`v1` moves when what executes or what is documented
changes**, and not otherwise. `chore` is deliberately hidden: tidying a comment should not push a
new version at every 434 site.

The trap is the flip side — a real behaviour change titled `chore:` cuts no release, appears in no
changelog, and sits on `main` reaching nobody. `pr-title-lint.yml` calls that out on the PR, but it
cannot read your mind. If a change alters what the workflows *do*, it is a `fix:` or a `feat:`.

One exception to "hidden never releases": a **breaking marker overrides the hidden list**.
`determineReleaseType` checks `commit.breaking` *before* it looks at the type, and the
BREAKING CHANGES section of the changelog is built from commit notes rather than from the type
sections. So `chore!: …` cuts a MAJOR, hidden or not.

#### Re-titling a dependabot PR

Dependabot always proposes `deps:`, which is a patch. Change the PR title before merging if that
is wrong — the squash commit takes the title, so this is the only edit needed.

| You want | Retitle to | Notes |
|---|---|---|
| PATCH | *(leave it)* | The default, and correct for almost every bump |
| MAJOR | `deps!: …` or `deps(github-actions)!: …` | `!` goes **after** the scope, before the colon. `deps!(): …` is not valid and the lint rejects it |
| MINOR | `feat(deps): …` | The only way — see below |

There is no `deps`-flavoured minor. Only the literal types `feat` and `feature` produce one
(`versioning-strategies/default.ts`), and no config option changes that. The other escape hatch,
a `Release-As: 1.2.0` footer, is awkward here: this repo's `squash_merge_commit_message` is
`COMMIT_MESSAGES`, so the PR *description* never reaches the commit — you would have to edit the
body in the squash-merge dialog.

In practice MINOR is close to an empty category for a pure dependency bump. A bump on its own
adds no input, secret or capability to *this* repo's interface, so it is a PATCH; if it could
break a site it is a MAJOR. It is only a MINOR when you also changed a workflow to expose
something the new version made possible — and that is your own `feat:` PR, with the bump riding
along in it.

#### If something goes wrong

- **No Release PR appeared.** Every commit since the last release is a hidden type. Check the
  workflow's run summary; the usual cause is a `chore:` or `ci:` title on a change that deserved
  `fix:`. Fix it by landing the next change with a releasing type, or force one with `Release-As:`.
- **`v1` points at the wrong commit.** Run
  [`update-major-tag.yml`](.github/workflows/update-major-tag.yml) via *Actions → Run workflow*
  and give it the `vX.Y.Z` tag `v1` should point at.
- **A version needs to be forced.** Add a `Release-As: 1.4.0` footer to a commit on `main`.

#### Configuration

| File | Purpose |
|---|---|
| [`.github/release-please-config.json`](.github/release-please-config.json) | Bump rules and changelog sections |
| [`.github/.release-please-manifest.json`](.github/.release-please-manifest.json) | Current version — **release-please owns this, do not hand-edit** |
| [`.github/VERSION`](.github/VERSION) | Same version as plain text, for humans and greps |

`last-release-sha` in the config pins the start of history to the `v1.1.0` commit, so the
pre-automation commits (`Initial commit`, `Update Readme`, …) are never scanned. Leave it.

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
