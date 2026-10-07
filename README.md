# 434 Marketing — Org-Wide GitHub Defaults

This repository contains shared GitHub Actions workflows and starter templates used across all [434 Marketing](https://434marketing.com) client site repositories.

## The deploy model

Three environments, three different triggers, and no two of them are the same event.

| Environment | Deploys when | Workflow in the site repo | Present on |
|---|---|---|---|
| **Dev** | every push to an open pull request | `dev.yaml` | only repos that have a dev install — normally an in-place rebuild |
| **Staging** | a pull request **merges** into the default branch, or a manual run from any branch | `stage.yaml` | every repo |
| **Production** | the release-please **Release PR merges**: release-please cuts the `vX.Y.Z` tag and calls `prod.yaml` with it. A `vX.Y.Z` tag pushed by hand also deploys | `release-please.yaml` + `prod.yaml` | every repo, gated off until launch |

Read as a sentence: **a pull request is dev, the default branch is staging, and a
release is production.** A branch is not an environment.

### What changed from v1, and why

v1 had two triggers and they were both one step too eager.

**Opening or updating a pull request deployed to staging.** Every push to any open
PR overwrote staging, so two open PRs meant staging showed whichever was pushed last
and nothing said so. In v2 the pull-request environment is **dev**, and staging is
what you get by merging. On a repo with no dev install a pull request now deploys
nowhere, which is the point: staging stops being scratch space.

**Merging a pull request deployed straight to production, and picked the version
afterwards.** `wpe-deploy-prod.yml` deployed on merge and *then* read `major` /
`minor` / `patch` labels off the PR to compute a tag. Landing code and publishing it
were the same keystroke, the version was decided after the deploy had already
happened, and a forgotten label silently shipped a patch bump. In v2 production
deploys a release, and the release comes from merging a Release PR that shows you the
exact version and changelog first.

### Why release-please *calls* the production deploy

A tag created with `GITHUB_TOKEN` does not start other workflows — only
`workflow_dispatch` and `repository_dispatch` are exempt from that rule. release-please
creates its tag with `GITHUB_TOKEN`, so a `push: tags` trigger in `prod.yaml` would
never fire for it. `release-please.yaml` therefore has a second job, `production`,
that runs only when a release was cut and calls `prod.yaml` with the new tag through
`workflow_call`. This repo moves its own `v<major>` tag the same way.

`prod.yaml` keeps `push: tags` for a tag pushed by hand, and `workflow_dispatch` to
re-deploy an existing tag. All three paths run the same plan job, so the trunk check
and the `PROD_DEPLOY_ENABLED` gate apply to each.

The consequence: **`release-please.yaml` needs `prod.yaml` in the same repo.** A
`uses:` pointing at a missing file fails the whole workflow at startup, Release PR
included. On a site that is not live yet, add `prod.yaml` anyway and uncomment its
gate (see [Setting up a site repo](#setting-up-a-site-repo)).

### If you want a PR branch on staging

That need is real and v2 keeps it — as a decision rather than a side effect:

```sh
gh workflow run stage.yaml --ref my-branch
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

One reusable workflow for every environment, and for themes, plugins and mu-plugins
alike. What differs between dev, staging and production is configuration, so it is
inputs rather than three files that have to be kept in step by hand.

```yaml
jobs:
  deploy:
    uses: 434marketing/.github/.github/workflows/wpe-deploy.yml@v2
    with:
      wpe_env: ${{ vars.WPE_STAGE_ENV }}
      src_path: themes/my-theme/
      remote_path: wp-content/themes/my-theme/
      lint_paths: themes/my-theme
      environment: staging
      backup: true
    secrets: inherit
```

The run is five jobs: **preflight** (refuses a dangerous deploy shape), **lint**,
**backup**, **deploy**, and a Slack notice on failure. Each one only starts when
everything before it succeeded or was deliberately skipped — a cancelled run never
reaches the rsync.

| Input | Required | Default | Notes |
|---|---|---|---|
| `wpe_env` | yes | — | Install **name**, not id |
| `environment` | yes | — | `dev` / `staging` / `production`. Created on first use; this is what gives you deployment history and the option of a required reviewer |
| `src_path` | one of these | — | A directory ends with `/` (its *contents* are copied); `./` is a repo whose root is the plugin. A single file is allowed too |
| `remote_path` | two, or `targets` | — | One folder: `wp-content/{plugins,themes,mu-plugins}/<slug>/`. For a file, the destination file |
| `targets` | | — | JSON list of `{src_path, remote_path}`, deployed in order after one lint and one backup. Replaces the two above. At most 6 |
| `allow_unsafe_remote_path` | no | `false` | The only bypass of the one-folder rule. Warns on every run |
| `ref` | no | caller's ref | Pass the merge commit on a merged `pull_request`. Resolved to one commit in preflight |
| `version` | no | `<branch>@<short-sha>` | A **label for the restore point**, never a release version |
| `backup` | no | `false` | Request a WP Engine restore point first |
| `backup_required` | no | `false` | No restore point, no deploy. `true` for production |
| `backup_wait` | no | `false` | Wait for `completed`. A 202 means *requested* |
| `lint_paths` | no | `.` | Space-separated `php -l` paths; empty disables the gate. **Set it** in any repo that holds more than the deployed code — `.` lints the whole repo |
| `php_version` | no | `8.2` | Match the install. Also the PHP for `setup_composer` |
| `php_versions` | no | — | JSON list, e.g. `["7.4","8.4"]`: lint once per version. Overrides `php_version` for lint |
| `rsync_flags` | no | see below | Replaces the whole string when set |
| `extra_excludes` | no | — | Space-separated patterns, appended as `--exclude=<p>`. Keeps the defaults |
| `build_command` | no | — | Runs from the repo root before the rsync, e.g. `composer install --no-dev` |
| `setup_node` | no | — | Node version for `build_command`, e.g. `20` |
| `setup_composer` | no | `false` | PHP + Composer for `build_command` |
| `post_deploy_script` | no | — | Script on the install, run after the last target. See [Post-deploy checks](#post-deploy-checks) |
| `smoke_wp_cli` | no | `false` | `wp eval` on the install after deploying; any fatal fails the run |
| `require_active_plugin` | no | — | Slug that must be active after deploying |
| `smoke_urls` | no | — | Space-separated paths fetched after deploying; non-2xx fails the run |
| `cache_clear` | no | `true` | Flush page and CDN cache, once, after the last target |
| `dry_run` | no | `false` | Preflight, lint, resolve the install, write nothing |

Only the deploy job declares the `environment`, so a **required reviewer** on it pauses
the run *after* the restore point is taken. The snapshot is still a valid restore point
for the code, which nothing has touched yet, but database and upload changes made while
the approval waits are not in it. Approve promptly, or take a fresh backup by
re-running.

**Required secret:** `WPE_SSHG_KEY_PRIVATE`

**Needed only when `backup: true`:** `WPE_API_USER_ID`, `WPE_API_PASSWORD` secrets and
the `BACKUP_NOTIFICATION_EMAIL` **variable**. `notification_emails` is a *required*
field on the WP Engine backup endpoint, so a missing or misspelled variable is a 400,
not a default.

**Optional secret:** `SLACK_WEBHOOK_URL` — posts when a deploy fails, and when a
**production** deploy is cancelled, because a cancellation mid-deploy is exactly when
someone needs to look.

**There is no `WPE_INSTALL_ID`, and there should not be.** v1's prod workflow held the
install's UUID as a secret, and an install id is per-install — so a repo could hold
exactly one, which is the whole reason v1 could back up production and never staging.
`wpe-deploy.yml` resolves the id from the install *name* through `GET /installs`, so
one org-level credential pair covers every install on every repo. If your account
does not have API access enabled it will see no installs at all; run with
`dry_run: true` to find that out before a deploy needs it. A WP Engine API error during
that lookup is reported as an error, never as "not found" — and on staging
(`backup_required: false`) it warns instead of blocking the deploy.

#### Preflight: what it refuses

The deploy action protects a site with one exclude list. Its static part —
`wp-config.php`, `_wpeprivate`, `.wpengine-conf/`, VCS files — applies to every deploy.
The part that guards `uploads/`, caches, drop-ins and 12 named WP Engine mu-plugins is
generated only when `REMOTE_PATH` is spelled *exactly* `''`, `.`, `wp-content(/)` or
`wp-content/mu-plugins(/)`. Nothing in it bounds `--delete`. Each of these exited
rsync 0 — a green deploy — when reproduced against the action's own code:

| `src_path` → `remote_path` | What happened |
|---|---|
| `plugins/lyh-welcome/` → `wp-content/plugins/` (slug forgotten) | Deleted every other plugin, and left this one's files loose at the plugins root |
| `plugins/` → `wp-content/plugins/` | Deleted every plugin not in git, and overwrote live third-party plugins with older git copies |
| `plugins/lyh-welcome` → `wp-content/plugins/lyh-welcome/` (no trailing slash) | Shipped into `lyh-welcome/lyh-welcome/`. The old plugin kept running |

So the **preflight** job checks every target before lint and backup, and again in the
deploy job after the build, against the tree that will actually ship. It fails the run
when:

1. `remote_path` starts with `/` or `./`, or contains `//`, a `.` or a `..` segment.
   The action's protections are an exact string match, so a non-canonical spelling
   silently switches them off.
2. `src_path` does not exist in the commit being deployed.
3. `src_path` is a directory without a trailing `/` (rsync would nest it one level
   too deep), or a file whose `remote_path` is not a file of the same name.
4. The flags contain `--delete` and `remote_path` is a shared root. For the site
   root, `wp-content/` and `wp-content/uploads/` there is no way through: deploy a
   folder, or drop `--delete`. For `wp-content/plugins/`, `themes/` or `mu-plugins/`,
   a root deploy you really mean needs `--filter='P /*'` in `rsync_flags`, placed
   before any include, `R` rule or rules file, *plus* `allow_unsafe_remote_path: true`.
   `P /*` keeps every top-level entry from being deleted. It protects one level only,
   which is why it cannot make the site root or `wp-content/` safe: everything one
   level down — each plugin, theme and upload — would still go. rsync applies the
   first rule that matches, so an earlier include would override it.
5. A directory deploy targets anything but **one** folder,
   `wp-content/{plugins,themes,mu-plugins}/<slug>/`, or a file deploy lands outside
   those three directories — unless `allow_unsafe_remote_path: true`, which warns on
   every run.
6. `src_path` looks one level too high — it contains a folder named like the
   destination slug, as `mu-plugins/` → `wp-content/mu-plugins/lyh-core/` would — or a
   `wp-content/plugins/<slug>/` folder has no `*.php` directly in it with a non-empty
   `Plugin Name:` header, or a `wp-content/themes/<slug>/` folder has no `style.css`
   with `Theme Name:`. That is almost always the wrong `src_path`.
7. A single `.php` file is deployed and `lint_paths` does not cover it. The action's own
   `PHP_LINT` runs `find "$SRC_PATH"/`, which finds nothing for a file and still prints
   success — so the lint job is the only lint that file gets.

It also rejects `--delete-excluded` (it would delete the very files the action's
excludes protect), `--inplace` together with `--delay-updates` (rsync refuses the pair),
`--relative` / `-R` and `--files-from` (they move or replace the source), two targets
with the same destination, nested targets under `--delete`, a non-option word in
`rsync_flags`, an empty `wpe_env` (an unset `WPE_*_ENV` variable), and malformed
`targets`, `php_versions`, `post_deploy_script`, `require_active_plugin` or `smoke_urls`
values. It warns when an mu-plugin loader is listed before its folder.

With `build_command` set, a `src_path` the build will create cannot be checked before the
build. Preflight then still applies every rule that depends only on `remote_path`, and
leaves the rest to the second pass after the build.

The run is also pinned to **one commit**: preflight resolves `ref` to a SHA, and lint,
the backup label and the deploy all use that SHA. A branch that moves during a
30-minute production backup wait cannot slip a different commit into the rsync.

#### Deploying plugins and mu-plugins

Every shape below is one `with:` block. The workflow templates carry the same
examples next to `THEME_NAME`.

**A plugin in a subfolder of the repo:**

```yaml
      src_path: plugins/lyh-welcome/
      remote_path: wp-content/plugins/lyh-welcome/
      lint_paths: plugins/lyh-welcome
```

**A repo whose root *is* the plugin** — `./` is the shape for that. Exclude what
should not ship; anchored patterns (a leading `/`) match only at the top of
`src_path`, so a vendored `vendor/foo/README.md` still ships:

```yaml
      src_path: ./
      remote_path: wp-content/plugins/lyh-welcome/
      lint_paths: .
      extra_excludes: /README.md /CHANGELOG.md /docs/ /tests/ /phpunit.xml.dist
```

**An mu-plugin** is two deploys, **in this order** — the folder, then the loader as a
single file — and never the `mu-plugins/` root, which preflight refuses under
`--delete`. A root deploy keeps only the 12 WP Engine mu-plugins the action's
exclude list happens to name, and deletes every other mu-plugin on the server: client
and vendor ones, and any WP Engine mu-plugin newer than that list.

```yaml
      targets: >-
        [{"src_path": "mu-plugins/lyh-core/",
          "remote_path": "wp-content/mu-plugins/lyh-core/"},
         {"src_path": "mu-plugins/lyh-core-loader.php",
          "remote_path": "wp-content/mu-plugins/lyh-core-loader.php"}]
      lint_paths: mu-plugins
```

Folder first, because WordPress runs every top-level `wp-content/mu-plugins/*.php` on
every request, so a loader that lands before its code is a site-wide fatal. And an
mu-plugin fatal is worse than a plugin one: **WordPress's recovery mode does not cover
mu-plugins.** Guard the loader so a missing folder degrades instead of killing the site:

```php
<?php
// wp-content/mu-plugins/lyh-core-loader.php
if ( is_readable( __DIR__ . '/lyh-core/lyh-core.php' ) ) {
	require_once __DIR__ . '/lyh-core/lyh-core.php';
}
```

**Theme + plugin + mu-plugin in one deploy.** One lint pass, **one** backup, then the
targets deploy one after another in array order, stopping at the first failure. The
cache is flushed once, after the last one. Three separate calls would mean three
backups (each up to 30 minutes on production), three cache flushes, and no order.

```yaml
      targets: >-
        [{"src_path": "themes/lyh/",            "remote_path": "wp-content/themes/lyh/"},
         {"src_path": "plugins/lyh-welcome/",   "remote_path": "wp-content/plugins/lyh-welcome/"},
         {"src_path": "mu-plugins/lyh-core/",   "remote_path": "wp-content/mu-plugins/lyh-core/"},
         {"src_path": "mu-plugins/lyh-core-loader.php",
          "remote_path": "wp-content/mu-plugins/lyh-core-loader.php"}]
      lint_paths: themes/lyh plugins/lyh-welcome mu-plugins
```

**A plugin with a build step.** CI deploys a *checkout*, so anything git-ignored —
`vendor/`, `build/`, compiled assets — is not in it, and `--delete` removes the
server's copy. A plugin with git-ignored runtime dependencies must either build in the
deploy job or drop `--delete` from `rsync_flags`:

```yaml
      src_path: plugins/lyh-welcome/
      remote_path: wp-content/plugins/lyh-welcome/
      setup_composer: true
      setup_node: "20"
      build_command: >-
        cd plugins/lyh-welcome && composer install --no-dev --optimize-autoloader
        && npm ci && npm run build
```

The build runs in the deploy job, after checkout and before the rsync, so the built
tree is what ships; preflight runs again after it. The lint job lints the source, not
the build output. Note that `node_modules` and `package.json` are in the default
excludes; `vendor/` is not.

**A plugin with a PHP support floor:** `php_versions: '["7.4","8.4"]'` lints against
each version in parallel.

#### Plugin lifecycle

- **The first deploy does not activate the plugin.** Run `wp plugin activate <slug>`
  once per install (and `--network` on multisite).
- **A renamed main file or folder deactivates the plugin, silently.** WordPress stores
  the active plugin as `folder/file.php`. Set `require_active_plugin` so the deploy
  fails instead.
- **To remove a plugin,** run `wp plugin deactivate <slug> && wp plugin delete <slug>`
  on each install *before* deleting the caller. Removing the workflow leaves the
  plugin on the server; nothing else ever deletes it.

#### Post-deploy checks

`php -l` cannot see a runtime fatal — a missing class, a bad `require_once`, a function
that does not exist on the install's PHP. A plugin fatal leaves the front end broken
and an mu-plugin fatal has no recovery mode, and without a check the run is green
either way. All four are opt-in and run after the last target, so a failure reaches
the Slack notice:

| Input | What it does |
|---|---|
| `smoke_wp_cli: true` | Runs `wp eval 'echo "ok";'` over the SSH gateway. That loads WordPress with every active plugin, mu-plugin and the theme, so any fatal in them fails the run |
| `require_active_plugin: lyh-welcome` | `wp plugin is-active lyh-welcome` must succeed |
| `smoke_urls: "/ /wp-login.php"` | Fetches each path from `https://<install>.wpenginepowered.com`, following redirects, retrying a 5xx twice; anything but a final 2xx fails |
| `post_deploy_script: wp-content/plugins/lyh-welcome/bin/post-deploy.sh` | Runs the script with `bash` on the install, from the site root, after the rsync. Non-zero fails the run |

```yaml
      smoke_wp_cli: true
      require_active_plugin: lyh-welcome
      smoke_urls: "/ /wp-login.php"
```

Things worth knowing:

- **These checks run after the code is live.** A failure means the bad code *is*
  deployed and the run says so; fix forward, or re-deploy the previous release
  (`gh workflow run prod.yaml -f tag=vX.Y.Z`).
- `smoke_wp_cli` runs the install's **CLI** PHP, which is its *configured* version.
  During a PHP Test Driver session the web tier can serve a different one, and only
  `smoke_urls` exercises that.
- The wp-cli checks send their script over stdin to `bash -s`. The WP Engine SSH
  gateway strips quoting from a command line, so the same commands passed as an `ssh`
  argument arrive mangled.
- A password-protected install answers `smoke_urls` with a 401. Don't use it there.
- `post_deploy_script` must be a path **inside a deployed folder**, given from the site
  root, so this deploy's rsync refreshes it — preflight checks both. The reason is an
  upstream bug: the action uploads the script only when it is *missing* on the server
  (`entrypoint.sh` tests `test -s` and an uninitialised `status`), so a script that
  already exists there is never updated by the action itself.

#### What this workflow will not do

- **Tag, or cut a GitHub Release.** Versioning belongs to release-please in the site
  repo. A deploy that also decides the version means the release cannot be reviewed
  before it ships.
- **Read `major` / `minor` / `patch` labels.** Version numbers come from commit
  messages. See [Versioning a site repo](#versioning-a-site-repo).
- **Activate plugins, run content migrations or flush rewrite rules.** Neither the
  action nor this workflow touches the database. Those stay manual.

#### rsync flags

The default is:

```
-azvr --delete --delay-updates --delete-delay --exclude=.* --exclude=node_modules
--exclude=package.json --exclude=package-lock.json --exclude=yarn.lock
--exclude=vite.config.js --exclude=webpack.mix.js --exclude=gulpfile.js
--exclude=postcss.config.js
```

Write every option as `--name=value`. Preflight rejects a word in `rsync_flags` that is
not an option, because rsync would read it as one more *source* and deploy it into the
target too — `--exclude=foo bar` merges a folder called `bar` into the plugin, green.
It also rejects an empty `rsync_flags`, which the action would quietly replace with its
own `--inplace` default.

Whatever `rsync_flags` says, the action **always appends**
`--exclude-from=<its generated list>` and `--chmod=D775,F664` after it
(`entrypoint.sh` in `wpengine/site-deploy`). The generated list is the WP Engine
protection described under [Preflight](#preflight-what-it-refuses), and it applies only
to the exact `REMOTE_PATH` spellings listed there.

The default differs from the deploy action's own (`-azvr --inplace --exclude=".*"`) in
three ways, and all are deliberate.

**`--delete` is added.** The action's default never deletes, so a file removed from
git lives on the install forever while the deploy still reports success. WordPress
discovers page templates by regex-scanning theme PHP files *on disk*, so a deleted
template keeps appearing in the block editor's Template panel — and if it is a
high-priority match like `front-page.php` it keeps *serving*. **What bounds `--delete`
is preflight**, which holds every directory deploy to a single plugin, theme or
mu-plugin folder. The action does not bound it.

**`--inplace` is removed**, which is what makes each file's replacement atomic. It had
been copied around the fleet unexamined — it is in the action's default, so it was in
both v1 workflows and in the action's own README example. It writes incoming bytes
straight into the live destination file, skipping rsync's default of building a
temporary file in the destination directory and `rename()`-ing it over the target when
complete. A same-directory `rename()` is atomic, so under the default a concurrent
reader gets the whole old file or the whole new one. rsync's manual, on `--inplace`:

> The file's data will be in an inconsistent state during the transfer and will be
> left that way if the transfer is interrupted or if an update fails.

A live WordPress site is the textbook in-use file set — `functions.php`, a plugin's
main file and every template are read off disk on each request — so a request landing
inside that window compiles a truncated or spliced file and fatals. With
`display_errors` off on WP Engine that is a blank page whose only explanation is the
install's error log. Opcache does not save you: revalidating inside the window can
compile the garbage and serve it until the next mtime change. And `--inplace` implies
`--partial`, so an interrupted or cancelled run *leaves* the broken bytes in place.

**`--delay-updates --delete-delay` are added.** Per-file atomicity alone still let a
new plugin main file go live before the new include it `require_once`s had arrived, and
every request in that window fatals — reproduced. With these two flags each updated
file waits in a `.~tmp~` holding directory, all of them are renamed into place in one
burst at the **end** of the transfer, and deletions happen after that.

**The honest limit:** each file is atomic and all updates are applied together at the
end of the transfer, but that burst of renames is not itself one atomic act — a request
can still land between two of them. In a 3,000-file harness run, 1,653 files were still
waiting when the first one went live. Only a build directory plus a symlink swap makes a
whole deploy atomic, and the WP Engine action cannot do that.

**What an interrupted deploy leaves behind** (reproduced with the action's own code and
rsync 3.4.3, killing either end with SIGTERM or SIGKILL): the live files are exactly as
they were, and nothing is deleted. Beyond that:

- If the deploying side dies — a cancelled run, a dropped connection — the folder keeps a
  `.~tmp~/` holding directory with the finished new files and the one that was in
  flight. The next successful deploy of the same or a later commit normally removes it.
  It stays for good only if a file waiting in it has since been deleted from git:
  rsync protects its own holding directory from `--delete`. Remove that one by hand.
- Only if the server-side rsync itself is killed is the in-flight file left as a stray
  `.name.XXXXXX` dotfile in the live directory. `--exclude=.*` shields it from
  `--delete`, so remove it by hand.

Compare the action's default: an interrupted `--inplace` run left the live file
matching neither the old nor the new version, with earlier files already new.

The `--inplace` fix shipped to every `@v1` site in **v1.1.3** (#16).

**Excludes.** To add one, use `extra_excludes` rather than restating `rsync_flags` —
the defaults then stay as they are:

```yaml
      extra_excludes: /README.md /CHANGELOG.md /docs/
```

- A leading `/` anchors a pattern to the top of `src_path`. Unanchored, `README.md`
  would also drop every vendored `vendor/*/README.md`.
- An exclude also **shields** the server's copy from `--delete`: an excluded path is
  neither updated nor removed.
- Git-ignored files never ship, whatever the excludes say — CI deploys a checkout.

**Dotfiles never deploy, and never update.** `--exclude=.*` means a plugin-shipped
`.htaccess` or `.well-known/` never reaches the server, and a stale server copy is
neither updated nor removed. To ship one, restate `rsync_flags` with an include placed
**before** `--exclude=.*` — rsync uses the first rule that matches:

```yaml
      rsync_flags: >-
        -azvr --delete --delay-updates --delete-delay
        --include=/.htaccess --exclude=.*
        --exclude=node_modules --exclude=package.json --exclude=package-lock.json
        --exclude=yarn.lock --exclude=vite.config.js --exclude=webpack.mix.js
        --exclude=gulpfile.js --exclude=postcss.config.js
```

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
them out of `workflow-templates/`. Replace `THEME_NAME` in each one — or swap in the
plugin or mu-plugin block from the comments beside it. Nothing else needs editing:
the templates find the repository's default branch themselves, so `main`, `master`
and `develop` repos all work as copied.

| Add | When |
|---|---|
| `stage.yaml` | always |
| `release-please.yaml` **and** `prod.yaml` | always, together — release-please calls `prod.yaml` when it cuts a release. **Not live yet?** Uncomment the `if: vars.PROD_DEPLOY_ENABLED == 'true'` line on `prod.yaml`'s plan job, and set that variable to `true` on launch day |
| `dev.yaml` | only if the site has a dev install |

`release-please.yaml` also needs three files that a workflow template cannot carry.
Copy them from `client-repo-templates/` and edit the paths in the config:

```sh
mkdir -p .github
cp client-repo-templates/release-please-config.json      .github/
cp client-repo-templates/.release-please-manifest.json   .github/
cp client-repo-templates/VERSION                         .github/
```

A repo that already has `vX.Y.Z` tags — every repo that ran v1's prod workflow — needs
one more step first: [Repos that already have release tags](#repos-that-already-have-release-tags).

Then set, per repo:

| | Kind | Value |
|---|---|---|
| `WPE_DEV_ENV` | variable | dev install name, if there is one |
| `WPE_STAGE_ENV` | variable | staging install name |
| `WPE_PROD_ENV` | variable | production install name |

`WPE_SSHG_KEY_PRIVATE`, `WPE_API_USER_ID`, `WPE_API_PASSWORD` and
`BACKUP_NOTIFICATION_EMAIL` live at the **organization** level and need nothing
per repo.

Keep the variables at repository or organization level, **not** on a GitHub
Environment. The callers read `WPE_*_ENV` in `with:` and the backup job reads
`BACKUP_NOTIFICATION_EMAIL` before any job that declares an environment runs, so an
environment-level variable arrives empty. If you restrict the `production`
environment to certain branches or tags, allow **both** the default branch and
`v*` tags: a release deploys from the release-please run on the default branch, and
a hand-pushed tag from the tag.

One repo setting: **Settings → Actions → General → "Allow GitHub Actions to create
and approve pull requests"** must be on, or release-please cannot open its PR.

The Release PR is opened by `github-actions[bot]`, so on a repo with `dev.yaml` its
pull-request run waits for **Approve workflows to run**. Leave it unapproved; the
Release PR changes only the changelog and version files.

And one label, used by `stage.yaml` to land something on the trunk without shipping it:

```sh
gh label create skip-deploy --color BFD4F2 --description "Merge without deploying"
```

### Rehearse before the first real deploy

`dry_run` runs preflight, lints, resolves the install name through the API and writes
nothing. It is the only way to find out that a credential pair cannot see an install
— or that a deploy shape is one preflight refuses — *before* a deploy needs it. Worth
one run per repo during the rollout. Both `stage.yaml` and `prod.yaml` expose it:

```sh
gh workflow run stage.yaml --ref main -f dry_run=true
gh workflow run prod.yaml -f tag=v1.4.2 -f dry_run=true
```

The production dry run gets through the NOT YET LIVE gate, which lets `dry_run`
pass, so a site can rehearse against its production install before launch.

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
4. The same run then calls `prod.yaml` with that tag, which deploys production.

Verified per type against the client config:

| Title | Effect |
|---|---|
| `feat:` | MINOR |
| `fix:`, `docs:`, `perf:`, `deps:`, `refactor:`, `revert:` | PATCH |
| `!` before the colon, or a `BREAKING CHANGE:` footer — any type, hidden or not | MAJOR |
| `chore:`, `ci:`, `build:`, `test:`, `style:` | nothing |
| a type the config does not list (`wip:`, `feature:`), or a title that is not a conventional commit (`Update style.css`, GitHub's default `Revert "…"`) | nothing — and not in the changelog |

One trap: `feature:` cuts nothing on its own, but next to a visible commit it silently
makes the release a MINOR, without appearing in the changelog. Write `feat:`. A
`Release-As: 1.5.0` footer forces an exact version, whatever the type.

The first release of a new repo is `1.0.0`, not `0.1.0` — with no prior release,
release-please skips the bump rules and returns `initial-version`, which is `1.0.0` for
release-type `simple`.

### The `Version:` header needs markers

Listing a file in `extra-files` does **not**, on its own, rewrite a theme's or plugin's
`Version:` header. release-please gives a string `extra-files` entry a format-specific
updater only for `.json`, `.yaml`/`.yml`, `.toml` and `.xml`; anything else — `.css`,
`.php` — gets the `Generic` updater, which rewrites only lines carrying
`x-release-please-version`, or lines between `x-release-please-start-version` and
`x-release-please-end`. An unmarked header never changes.

Wrap the header line in markers on **their own lines**, around the `Version:` line
**only**. WordPress's `get_file_data()` reads a header to the end of its line, so a
marker on the `Version:` line itself shows up in **Appearance → Themes** as part of the
version (`1.3.0 x-release-please-version` — checked against WordPress). And inside a
start/end block the updater rewrites the first version-looking string on *every* line,
so a block that also covers `Tested up to: 6.4.2` turns that into the release version
too:

```css
/*
Theme Name: LYH
x-release-please-start-version
Version: 1.2.3
x-release-please-end
*/
```

```php
<?php
/**
 * Plugin Name:       LYH Welcome
 * x-release-please-start-version
 * Version:           1.2.3
 * x-release-please-end
 */

define( 'LYH_WELCOME_VERSION', '1.2.3' ); // x-release-please-version
```

An inline marker is fine on a PHP constant, where nothing parses the rest of the line.
With markers in place, what WordPress shows matches the tag — which is also why the
release commit is worth deploying to staging rather than skipping.

The template config lists `themes/THEME_NAME/style.css`. For a plugin, list its main
file instead, or as well — `extra-files` takes several paths:

```json
"extra-files": ["themes/THEME_NAME/style.css", "plugins/PLUGIN_SLUG/PLUGIN_SLUG.php"]
```

The example lives here rather than in the template because release-please reads its
config with plain `JSON.parse`, so a comment in that file fails the run. A path that
does not exist — `THEME_NAME` left unedited, or `VERSION` not copied — does **not**
fail it: release-please logs a warning and skips the file. Check that the first Release
PR's diff touches the header file and `.github/VERSION`.

### Versioning in a mixed repo

When the deployable plugin is one folder in a repo that holds other things, scope
release-please to that folder, so only commits touching it count:

```json
{
  "release-type": "simple",
  "include-v-in-tag": true,
  "packages": {
    "plugins/lyh-welcome": {
      "include-component-in-tag": false,
      "changelog-path": "CHANGELOG.md",
      "version-file": "VERSION",
      "extra-files": ["lyh-welcome.php"]
    }
  }
}
```

with `.github/.release-please-manifest.json` set to `{ "plugins/lyh-welcome": "1.2.3" }`.
The example shows only what changes. Keep everything else from the template config —
`changelog-sections` especially: without it release-please falls back to its own
defaults, which hide `docs` and `refactor` and drop `deps`, so those stop cutting
releases. The template's `release-please.yaml` reads the package's prefixed outputs
(`plugins/lyh-welcome--tag_name`) itself, so it needs no edit.

- Only commits that touch a file under `plugins/lyh-welcome/` count toward its version
  and changelog. A `feat:` that touches only the theme, or a sibling folder such as
  `plugins/lyh-welcome-pro/`, does not.
- `changelog-path`, `version-file` and `extra-files` resolve **relative to the package
  folder**; prefix one with `/` to make it relative to the repo root. The package
  needs its own `VERSION` file.
- `include-component-in-tag: false` keeps the tags `vX.Y.Z`, which is what `prod.yaml`
  deploys. Set to `true` — or misspelled, since release-please silently ignores an
  unknown key — the tag becomes `<component>-vX.Y.Z`, and `prod.yaml` refuses it.

### Repos that already have release tags

Every repo that ran v1's prod workflow has `vX.Y.Z` tags and GitHub Releases. Copied
unchanged, the template manifest (`0.0.0`) makes release-please propose `1.0.0` with the
whole history in its changelog — a version that already exists. The first step of
`release-please.yaml` refuses to run in that state, so this is the fix it points at:

1. Find the newest tag: `git fetch --tags && git tag -l 'v*' --sort=-v:refname | head -1`.
2. Set `.github/.release-please-manifest.json` to that version without the `v` —
   `{ ".": "1.4.2" }` — and `.github/VERSION` to `1.4.2`.
3. Replace `prod.yaml` with the v2 template **in the same PR**. v1's prod workflow tags
   every merge; if it tags this one, the manifest is behind again. After the PR merges,
   repeat step 1, and if there is a newer tag, update the manifest to it before merging
   any Release PR. The guard step will say so if you forget.

The first Release PR then proposes the next version — `1.4.3` or `1.5.0` — and lists
only the commits since `v1.4.2`. release-please finds `v1.4.2` by its tag name, so the
v1-created Release (or a bare tag) is enough.

Do **not** use `bootstrap-sha` or `initial-version` for this: the first still proposes
`1.0.0`, and the second ignores `feat`/`fix` and re-lists the whole history. If the
newest tag's commit is not in the default branch's history
(`git merge-base --is-ancestor v1.4.2 origin/main` fails), also set top-level
`"last-release-sha"` to `git rev-list -n1 v1.4.2`, or the changelog re-lists everything.

The rest of the mechanics — which types release and why, how to retitle a dependabot
PR, the phantom-commit trap in PR descriptions — is the same as for this repo and is
documented under [Cutting a release](#cutting-a-release) below.

## Migrating from v1 to v2

Nothing breaks until you do this. `@v1` keeps working.

Per repo, roughly ten minutes:

1. **Add `release-please.yaml` and the release-please config files, and replace
   `prod.yaml` with the v2 template, in one PR.** Do this first, and set the manifest
   to the repo's newest v1 tag (see
   [Repos that already have release tags](#repos-that-already-have-release-tags)).
   They must land together: `release-please.yaml` calls `prod.yaml`, and a v1
   `prod.yaml` has no `workflow_call` trigger. Production in v2 deploys a release, so
   without this there is nothing to deploy.
2. **Replace `stage.yaml`** with the v2 template. Note the trigger change: staging
   now deploys on merge, not on PR update.
3. **Delete the `major` / `minor` / `patch` labels** from the repo. The v2 `prod.yaml`
   from step 1 overwrote the v1 one at the same path and reads no labels, so nobody
   should keep applying them expecting an effect.
4. **Add `dev.yaml`** if the site has a dev install.
5. **Remove the `WPE_INSTALL_ID` secret.** v2 resolves install ids by name.
6. **Add the `WPE_DEV_ENV` variable** if you added `dev.yaml`.
7. **Rehearse:** `gh workflow run stage.yaml --ref <default-branch> -f dry_run=true`.

Then verify: open a throwaway PR and confirm it deploys to dev (or nowhere), merge it
and confirm staging updates, then merge the Release PR and confirm the same
release-please run shows a `production` job that deploys the new tag.

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
