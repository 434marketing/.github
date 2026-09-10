# 434 Marketing — Org-Wide GitHub Defaults

This repository contains shared GitHub Actions workflows and starter templates used across all [434 Marketing](https://434marketing.com) client site repositories.

## The deploy model

Three environments, three different triggers, and no two of them are the same event.

| Environment | Deploys when | Workflow in the site repo | Present on |
|---|---|---|---|
| **Dev** | every push to an open pull request | `dev.yaml` | only repos that have a dev install — normally an in-place rebuild |
| **Staging** | a pull request **merges** into the default branch, or a manual run from any branch | `stage.yaml` | every repo |
| **Production** | a `vX.Y.Z` **tag** is pushed — which happens when the release-please Release PR is merged | `prod.yaml` + `release-please.yaml` | every live repo |

Read as a sentence: **a pull request is dev, the default branch is staging, and a
tag is production.** A branch is not an environment.

### What changed from v1, and why

v1 had two triggers and they were both one step too eager.

**Opening or updating a pull request deployed to staging.** Every push to any open
PR overwrote the staging theme, so two open PRs meant staging showed whichever was
pushed last and nothing said so. In v2 the pull-request environment is **dev**, and
staging is what you get by merging. On a repo with no dev install a pull request now
deploys nowhere, which is the point: staging stops being scratch space.

**Merging a pull request deployed straight to production, and picked the version
afterwards.** `wpe-deploy-prod.yml` deployed on merge and *then* read `major` /
`minor` / `patch` labels off the PR to compute a tag. Landing code and publishing it
were the same keystroke, the version was decided after the deploy had already
happened, and a forgotten label silently shipped a patch bump. In v2 production
deploys from a tag, and the tag comes from merging a Release PR that shows you the
exact version and changelog first.

### If you want a PR branch on staging

That need is real and v2 keeps it — as a decision rather than a side effect:

```sh
gh workflow run stage.yml --ref my-branch
```

or **Actions → Deploy to WP Engine Staging → Run workflow**, and pick the branch.
`workflow_dispatch` runs the workflow from the ref you choose, and the run records
who asked for it.

If you would rather have the old convenience on a specific repo — say a site with no
dev install where you are the only person pushing — widen one line in that repo's
`stage.yaml`:

```yaml
on:
  pull_request:
    types: [closed, opened, synchronize, reopened]
```

The gate job already handles both shapes, so that is the entire change. It warns in
the run summary when it fires from an open PR, so the choice stays visible.

## Reusable Workflows

### `wpe-deploy.yml` — **v2, use this one**

One reusable workflow for every environment. What differs between dev, staging and
production is configuration, so it is inputs rather than three files that have to be
kept in step by hand.

```yaml
jobs:
  deploy:
    uses: 434marketing/.github/.github/workflows/wpe-deploy.yml@v2
    with:
      wpe_env: ${{ vars.WPE_STAGE_ENV }}
      src_path: themes/my-theme/
      remote_path: wp-content/themes/my-theme/
      environment: staging
      backup: true
    secrets: inherit
```

| Input | Required | Default | Notes |
|---|---|---|---|
| `wpe_env` | yes | — | Install **name**, not id |
| `src_path` | yes | — | Trailing slash copies the directory's *contents* |
| `remote_path` | yes | — | Destination on the install |
| `environment` | yes | — | `dev` / `staging` / `production`. Created on first use; this is what gives you deployment history and the option of a required reviewer |
| `ref` | no | caller's ref | Pass the merge commit on a merged `pull_request` |
| `version` | no | `<ref-name>@<short-sha>` | A **label for the restore point**, never a release version |
| `backup` | no | `false` | Request a WP Engine restore point first |
| `backup_required` | no | `false` | No restore point, no deploy. `true` for production |
| `backup_wait` | no | `false` | Wait for `completed`. A 202 means *requested* |
| `lint_paths` | no | `.` | Space-separated `php -l` paths; empty disables the gate |
| `php_version` | no | `8.2` | Match the install |
| `rsync_flags` | no | see below | Replaces the whole string when set |
| `cache_clear` | no | `true` | Flush page and CDN cache after deploying |
| `dry_run` | no | `false` | Lint, resolve the install, write nothing |

**Required secret:** `WPE_SSHG_KEY_PRIVATE`

**Needed only when `backup: true`:** `WPE_API_USER_ID`, `WPE_API_PASSWORD` secrets and
the `BACKUP_NOTIFICATION_EMAIL` **variable**. `notification_emails` is a *required*
field on the WP Engine backup endpoint, so a missing or misspelled variable is a 400,
not a default.

**Optional secret:** `SLACK_WEBHOOK_URL` — a failed deploy posts to it if it is set.

**There is no `WPE_INSTALL_ID`, and there should not be.** v1's prod workflow held the
install's UUID as a secret, and an install id is per-install — so a repo could hold
exactly one, which is the whole reason v1 could back up production and never staging.
`wpe-deploy.yml` resolves the id from the install *name* through `GET /installs`, so
one org-level credential pair covers every install on every repo. If your account
does not have API access enabled it will see no installs at all; run with
`dry_run: true` to find that out before a deploy needs it.

#### What this workflow will not do

- **Tag, or cut a GitHub Release.** Versioning belongs to release-please in the site
  repo. A deploy that also decides the version means the release cannot be reviewed
  before it ships.
- **Read `major` / `minor` / `patch` labels.** Version numbers come from commit
  messages. See [Versioning a site repo](#versioning-a-site-repo).
- **Run content migrations or flush rewrite rules.** Neither the action nor this
  workflow touches the database. Those stay manual.

#### rsync flags

The default is:

```
-azvr --delete --exclude=.* --exclude=node_modules --exclude=package.json
--exclude=package-lock.json --exclude=yarn.lock --exclude=vite.config.js
--exclude=webpack.mix.js --exclude=gulpfile.js --exclude=postcss.config.js
```

It differs from the deploy action's own default (`-azvr --inplace --exclude=".*"`) in
two ways, and both are deliberate.

**`--delete` is added.** The action's default never deletes, so a file removed from
git lives on the install forever while the deploy still reports success. WordPress
discovers page templates by regex-scanning theme PHP files *on disk*, so a deleted
template keeps appearing in the block editor's Template panel — and if it is a
high-priority match like `front-page.php` it keeps *serving*. Blast radius is bounded
by `remote_path`, and rsync never deletes a path matching an `--exclude`, so dotfiles
on the install survive.

**`--inplace` is removed**, which is what makes each file's replacement atomic. It had
been copied around the fleet unexamined — it is in the action's default, so it was in
both v1 workflows and in the action's own README example. It writes incoming bytes
straight into the live destination file, skipping rsync's default of building a
temporary file in the destination directory and `rename()`-ing it over the target when
complete. A same-directory `rename()` is atomic, so under the default a concurrent
reader gets the whole old file or the whole new one. rsync's manual, on `--inplace`:

> The file's data will be in an inconsistent state during the transfer and will be
> left that way if the transfer is interrupted or if an update fails.

A live WordPress theme is the textbook in-use file — `functions.php` and every
template are read off disk on each request — so a request landing inside that window
compiles a truncated or spliced file and fatals. With `display_errors` off on WP
Engine that is a blank page whose only explanation is the install's error log.
Opcache does not save you: revalidating inside the window can compile the garbage and
serve it until the next mtime change. And `--inplace` implies `--partial`, so an
interrupted or cancelled run *leaves* the broken bytes in place.

**The honest limit:** this makes each *file* atomic, not the *deploy*. Halfway through
an rsync, one template can be new while another is still old. It moves the failure
mode from "a file is syntactically invalid" to "two valid files are briefly from
different commits". Real deploy atomicity needs a build directory and a symlink swap,
which the WP Engine action cannot do. One residual: without `--inplace`, an rsync
killed with `SIGKILL` can leave a `.name.XXXXXX` temp file behind, and `--exclude=.*`
means `--delete` will never clean it up — a stray dotfile rather than a served
template, so still the better trade.

This fix is also backported to the two v1 workflows below, so it reaches `@v1` sites
as soon as a `v1.x` release is cut.

---

### `wpe-deploy-staging.yml` and `wpe-deploy-prod.yml` — **v1, deprecated**

Kept unchanged in interface so that nothing pinned to `@v1` breaks. Bug fixes still
land in them; features do not. See [Migrating from v1 to v2](#migrating-from-v1-to-v2).

| | `wpe-deploy-staging.yml` | `wpe-deploy-prod.yml` |
|---|---|---|
| Inputs | `theme_path`, `remote_path` | same |
| Secrets | `WPE_SSHG_KEY_PRIVATE` | + `WPE_API_USER_ID`, `WPE_API_PASSWORD`, `WPE_INSTALL_ID` |
| Variables | `WPE_STAGE_ENV` | `WPE_PROD_ENV`, `BACKUP_NOTIFICATION_EMAIL` |
| Also does | — | tags and releases the **calling** repo from `major`/`minor`/`patch` PR labels |

> The variable is `BACKUP_NOTIFICATION_EMAIL`. This README said
> `BACKUP_EMAIL_NOTIFICATION` until v2 — the workflow has always read the former
> (fixed in `08d0020`). If you set the name this README used to give, unset it.

## Setting up a site repo

Add the workflows from **Actions → New workflow**, under *By 434marketing*, or copy
them out of `workflow-templates/`. Replace `THEME_NAME` in each one.

| Add | When |
|---|---|
| `stage.yaml` | always |
| `release-please.yaml` | always — `prod.yaml` needs the tag it creates |
| `prod.yaml` | once the site is live |
| `dev.yaml` | only if the site has a dev install |

`release-please.yaml` also needs two files that a workflow template cannot carry.
Copy them from `client-repo-templates/` and edit the theme path in the config:

```sh
mkdir -p .github
cp client-repo-templates/release-please-config.json      .github/
cp client-repo-templates/.release-please-manifest.json   .github/
cp client-repo-templates/VERSION                         .github/
```

Then set, per repo:

| | Kind | Value |
|---|---|---|
| `WPE_DEV_ENV` | variable | dev install name, if there is one |
| `WPE_STAGE_ENV` | variable | staging install name |
| `WPE_PROD_ENV` | variable | production install name |

`WPE_SSHG_KEY_PRIVATE`, `WPE_API_USER_ID`, `WPE_API_PASSWORD` and
`BACKUP_NOTIFICATION_EMAIL` live at the **organization** level and need nothing
per repo.

One repo setting: **Settings → Actions → General → "Allow GitHub Actions to create
and approve pull requests"** must be on, or release-please cannot open its PR.

And one label, used by `stage.yaml` to land something on the trunk without shipping it:

```sh
gh label create skip-deploy --color BFD4F2 --description "Merge without deploying"
```

### Rehearse before the first real deploy

`dry_run` lints, resolves the install name through the API and writes nothing. It is
the only way to find out that a credential pair cannot see an install *before* a
deploy needs it — worth one run per repo during the rollout.

```sh
gh workflow run stage.yml --ref main -f dry_run=true   # if you expose the input
```

## Versioning a site repo

Site repos version themselves with release-please, the same way this repo does. The
short version:

1. Merge a PR whose **title is a conventional commit**. Squash merging makes that
   title the commit subject, and the commit subject is the only thing release-please
   reads.
2. release-please keeps one open PR titled `chore(main): release X.Y.Z` showing the
   version and changelog it will write.
3. Merging that PR writes `CHANGELOG.md`, bumps the version, and creates the
   `vX.Y.Z` tag.
4. The tag triggers `prod.yaml`, which deploys production.

`feat:` cuts a minor, `fix:` a patch, a `!` before the colon cuts a major, and
`chore:` / `ci:` / `build:` / `test:` / `style:` cut nothing. The first release of a
new repo is `1.0.0`, not `0.1.0` — with no prior release, release-please skips the
bump rules and returns `initial-version`, which is `1.0.0` for release-type `simple`.

Listing `themes/THEME_NAME/style.css` in `extra-files` makes the release commit
rewrite the theme's `Version:` header, so what WordPress shows in **Appearance →
Themes** matches the tag. That is also why the release commit is worth deploying to
staging rather than skipping.

The rest of the mechanics — which types release and why, how to retitle a dependabot
PR, the phantom-commit trap in PR descriptions — is the same as for this repo and is
documented under [Cutting a release](#cutting-a-release) below.

## Migrating from v1 to v2

Nothing breaks until you do this. `@v1` keeps working.

Per repo, roughly ten minutes:

1. **Add `release-please.yaml` and its two config files, and let it open a Release
   PR.** Do this first and on its own. Production in v2 deploys from a tag, so
   without this there is nothing to deploy from.
2. **Replace `stage.yaml`** with the v2 template. Note the trigger change: staging
   now deploys on merge, not on PR update.
3. **Replace `prod.yaml`** with the v2 template. Delete the `major` / `minor` /
   `patch` labels from the repo afterwards so nobody keeps applying them expecting
   an effect.
4. **Add `dev.yaml`** if the site has a dev install.
5. **Remove the `WPE_INSTALL_ID` secret.** v2 resolves install ids by name.
6. **Add the `WPE_DEV_ENV` variable** if you added `dev.yaml`.

Then verify: open a throwaway PR and confirm it deploys to dev (or nowhere), merge it
and confirm staging updates, and merge the Release PR and confirm the tag deploys
production.

The v1 workflows stay for a reasonable window. They will be removed in `v3`, not
before, and not while anything still pins `v1`.

## Versioning this repo

This section is about `434marketing/.github` itself. For a client site repo, see
[Versioning a site repo](#versioning-a-site-repo).

Site repos reference these workflows **by tag, never by branch**. `v2` is current;
`v1` is still published and still receives fixes.

| Reference | Behavior | Use when |
|---|---|---|
| `@v2` | **Default.** Tracks the newest `v2.x.y`. Fixes and features arrive automatically; breaking changes never do. | Almost always |
| `@v1` | Same contract, previous major. Fixes only. | A site that has not been migrated yet |
| `@v2.1.3` | Frozen at one release. Nothing reaches the site until someone bumps it by hand. | A site is mid-migration, or you need a byte-reproducible deploy |
| `@main` | Tracks every commit as it lands. | Never — this is what tagging replaced |

### What counts as a breaking change

| Bump | Meaning |
|---|---|
| **MAJOR** (`v3`) | Renamed or removed inputs, a new required secret or variable, or any change that could break a site working today |
| **MINOR** (`v2.x`) | New optional inputs, new capability, or a changed default that is safe for every site |
| **PATCH** (`v2.0.x`) | Bug fixes with no interface change |

Sites on `@v2` inherit MINOR and PATCH **automatically**. That is the point of the floating
tag — but it means the bar for MAJOR is *"could this break a site that works today"*, not
*"does this feel like a big change"*. When in doubt, cut a major.

### Cutting a release

Releases are automatic. You never tag this repo by hand.

[`release-please.yml`](.github/workflows/release-please.yml) runs on every push to `main` and
keeps **one open Release PR** that accumulates the pending changes and shows the version they
will cut. Merging that PR is the act of releasing: it writes `CHANGELOG.md`, bumps
[`.github/VERSION`](.github/VERSION), creates the `vX.Y.Z` tag and the GitHub Release, and then
force-moves the matching `v<major>` tag to it.

So the whole procedure is:

1. Merge a PR into `main` with a **conventional commit title** (see below).
2. Merge the Release PR when you want to ship. Everything else happens on its own.

`main` allows squash and rebase merges only, and squash is the normal path — which means the
**PR title becomes the commit message**, and that title is the only thing release-please reads.
[`pr-title-lint.yml`](.github/workflows/pr-title-lint.yml) rejects a PR whose title will not
parse, and its check summary tells you which bump the title is about to cause.

#### Writing the title

| Title | Bump | Reaches `@v2` sites |
|---|---|---|
| `feat!: require a WPE_INSTALL_ID secret` | MAJOR → `v3.0.0` | **Never** — each site must update its `uses:` line |
| `feat: add an optional php_lint input` | MINOR → `v2.1.0` | Automatically |
| `fix: correct the rsync exclude for lockfiles` | PATCH → `v2.0.1` | Automatically |
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
*is* the release policy. It is set so that **the `v<major>` tag moves when what executes or what
is documented changes**, and not otherwise. `chore` is deliberately hidden: tidying a comment should not push a
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
(`versioning-strategies/default.ts`), and no config option changes that. The other escape hatch is
a `Release-As:` footer in the PR description, which forces an exact version and is checked before
anything else.

In practice MINOR is close to an empty category for a pure dependency bump. A bump on its own
adds no input, secret or capability to *this* repo's interface, so it is a PATCH; if it could
break a site it is a MAJOR. It is only a MINOR when you also changed a workflow to expose
something the new version made possible — and that is your own `feat:` PR, with the bump riding
along in it.

### The PR description is part of the commit message

This repo squash-merges with **"Pull request title and description"**, so the description you
write becomes the body of the commit on `main` — and release-please parses that body, not just
the title.

That is what makes the `Release-As:` footer above work. It also means a PR description can
accidentally create a **phantom commit**. release-please splits one commit message into several
wherever a blank line is immediately followed by a conventional-commit prefix
([`commit.ts`](https://github.com/googleapis/release-please/blob/main/src/commit.ts),
`splitMessages`):

```
feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert
```

So in a repo whose pull requests are frequently *about* versioning, a description like this one
cuts a MINOR from a docs-only PR, because the second paragraph parses as a real commit:

<pre>
Explains the release types.

feat: add an optional php_lint input
</pre>

The rule is narrow and easy to live with: **never start a paragraph with `type: `.** Keep every
example inside a fenced code block, a table cell, or inline backticks — a fence works because the
line above it is not blank. The same applies to a `BREAKING CHANGE:` footer, which cuts a MAJOR
from any PR whose description mentions one at the start of a line.

Note that `deps` is *not* in the split list, so a `deps: …` example in a description is harmless.

#### If something goes wrong

- **No Release PR appeared.** Every commit since the last release is a hidden type. Check the
  workflow's run summary; the usual cause is a `chore:` or `ci:` title on a change that deserved
  `fix:`. Fix it by landing the next change with a releasing type, or force one with `Release-As:`.
- **`v1` or `v2` points at the wrong commit.** Run
  [`update-major-tag.yml`](.github/workflows/update-major-tag.yml) via *Actions → Run workflow*
  and give it the `vX.Y.Z` tag that major should point at. It derives the major from the tag you
  give it, so it repairs either one.
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

Breaking changes ship as the next major and reach nobody until each site's `uses:` line is
updated. Keep shipping fixes to the previous major for a reasonable window so sites are not
forced to migrate on a deadline they did not choose — `v1` is in that window now, and the
per-repo steps are under [Migrating from v1 to v2](#migrating-from-v1-to-v2).

### A note on trust

Every site passes `secrets: inherit`, so these workflows receive that site's
`WPE_SSHG_KEY_PRIVATE` and WP Engine API credentials. Anyone able to merge here — or to move a
`v<major>` tag — can reach the deploy credentials of every 434 site. That is why `main` requires
a PR and an approving review, and why those rules should not be relaxed for convenience.

For a site where even that is too much trust, pin to a full commit SHA. A tag can be moved;
a SHA cannot:

```yaml
uses: 434marketing/.github/.github/workflows/wpe-deploy.yml@08d0020e907e0f3e849f0c48cc1a9df3a69dd307
```
