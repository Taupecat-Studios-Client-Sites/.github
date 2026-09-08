# Deploy to Pantheon

A reusable GitHub Actions workflow for deploying WordPress sites to [Pantheon](https://pantheon.io/).

## Usage

In your downstream repo's workflow file:

```yaml
jobs:
  deploy:
    uses: <org>/<this-repo>/.github/workflows/deploy.yml@v1
    with:
      pantheon_branch: master
    secrets: inherit
```

Pantheon allows one `sync_code` workflow per site at a time, so also add a `concurrency` block in the calling workflow if several of its own jobs can deploy at once. The reusable workflow already serializes per site and branch on its side.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pantheon_branch` | Yes | — | Pantheon Git branch to deploy to (e.g. `master`, `dev`) |
| `deploy_to_test` | No | `false` | Promote to Pantheon Test after pushing to Dev |
| `build_frontend` | No | `true` | Run the Node/npm build step |
| `web_docroot` | No | `true` | Whether the site uses a `web/` subdirectory as the Pantheon document root |
| `src_dir` | No | `web/wp-content/themes/<THEME_DIR>/src` | Path to the directory containing `package.json` and optionally `.nvmrc` |
| `extra_exclusions` | No | — | Additional rsync exclusions, one per line, added on top of the defaults (`.DS_Store`, `wp-content/uploads`) |
| `legacy_peer_deps` | No | `false` | Pass `--legacy-peer-deps` to `npm ci` |
| `git_user_name` | No | `github-actions[bot]` | Author name for the commit pushed to Pantheon |
| `git_user_email` | No | `41898282+github-actions[bot]@users.noreply.github.com` | Author email for the commit pushed to Pantheon |

## Secrets & Variables

The calling repo must have the following configured:

**Secrets:** `PANTHEON_SSH_KEY`, `PANTHEON_MACHINE_TOKEN`, `AUTH_JSON`, `ENV`, `ACTION_MONITORING_SLACK`

**Variables:** `PANTHEON_SITE_NAME`, `THEME_DIR` (required unless `src_dir` is passed or `build_frontend` is `false`)

## What It Does

1. Clones the Pantheon Git repo (via Terminus) in the background while the GitHub repo checks out in parallel.
2. Runs `composer install --no-dev` using `auth.json` and `.env` from secrets, installing Subversion first only if a package actually needs it.
3. If `build_frontend` is true: detects the Node version from `.nvmrc` (falls back to `lts/*`), runs `npm ci` (optionally with `--legacy-peer-deps`) and `npm run build`.
4. Rsyncs `web/`, `pantheon.yml`, and `wp-cli.yml` to the Pantheon local clone (into a `web/` subdirectory, or flattened to the repo root if `web_docroot` is `false`), excluding the default paths plus any listed in `extra_exclusions` and the theme `src/` directory.
5. Commits and pushes to the Pantheon branch. The commit message is the source commit's subject plus `Source-Repo`/`Source-Ref`/`Source-Commit`/`Source-Run` git trailers, so the originating GitHub commit stays recoverable from the Pantheon side.
6. Optionally promotes to Pantheon Test via `terminus env:deploy`.
7. Posts a Slack failure notification on any job failure.

Deploys are serialized per Pantheon site and branch (`concurrency`), because Pantheon runs only one `sync_code` workflow per site at a time.

## Reading deploy provenance on Pantheon

A Pantheon repo is not a mirror of the GitHub repo — this workflow commits a built artifact into Pantheon's own history, so the Pantheon commit SHA does not exist on GitHub. To recover the source commit from a Pantheon checkout (or a Quicksilver hook):

```bash
git log -1 --pretty='%(trailers:key=Source-Commit,valueonly=true)'
```

## Calling Repo Requirements

- Composer dependencies in the repo root (a `composer.json`).
- A theme `src/` directory with `package.json` (and optionally `.nvmrc`) if `build_frontend` is `true`.

## Versioning

**Tags live in this repo. The pin is written in the site's repo and resolves against this one.**

```yaml
# in the SITE's repo, e.g. .github/workflows/deploy.yml
    uses: <org>/<this-repo>/.github/workflows/deploy.yml@v1
    #     └──── this repo ─────┘                         └─ a tag in THIS repo
```

So releases are cut here, once, and each site gets a one-line `@ref` edit. A site never tags anything for this — its own tags and releases, if it has them, are unrelated.

Releases here are tagged `vX.Y.Z`, and a floating `vX` alias is moved forward to the newest release in that major line.

| Pin | Behavior |
|-----|----------|
| `@v1` | **Recommended.** Picks up fixes and new optional inputs; never a breaking change. |
| `@v1.2.3` | Fully reproducible. Use where a deploy must not change until someone bumps it. |
| `@main` | Unpinned — every merge here ships to your site on the next deploy, with no release notes. Avoid. |

A major bump means one of: an input removed or renamed, an optional input made required, a new required secret or repo variable, or a changed default that alters how an existing caller deploys. New optional inputs and internal changes that leave deploy behavior intact are minor or patch.

If a site currently uses `@main`, switch it to `@v1` — `@main` keeps working, but it tracks unreleased work.
