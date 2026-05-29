# Deploy to Pantheon

A reusable GitHub Actions workflow for deploying WordPress sites to [Pantheon](https://pantheon.io/).

## Usage

In your downstream repo's workflow file:

```yaml
jobs:
  deploy:
    uses: <org>/<this-repo>/.github/workflows/deploy.yml@main
    with:
      pantheon_branch: master
    secrets: inherit
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `pantheon_branch` | Yes | — | Pantheon Git branch to deploy to (e.g. `master`, `dev`) |
| `deploy_to_test` | No | `false` | Promote to Pantheon Test after pushing to Dev |
| `build_frontend` | No | `true` | Run the Node/npm build step |
| `src_dir` | No | `web/wp-content/themes/<THEME_DIR>/src` | Path to the directory containing `package.json` and optionally `.nvmrc` |

## Secrets & Variables

The calling repo must have the following configured:

**Secrets:** `PANTHEON_SSH_KEY`, `PANTHEON_MACHINE_TOKEN`, `AUTH_JSON`, `ENV`, `ACTION_MONITORING_SLACK`

**Variables:** `PANTHEON_SITE_NAME`, `THEME_DIR` (required unless `src_dir` is passed or `build_frontend` is `false`)

## What It Does

1. Clones the Pantheon Git repo (via Terminus) in the background while the GitHub repo checks out in parallel.
2. Runs `composer install --no-dev` using `auth.json` and `.env` from secrets.
3. If `build_frontend` is true: detects the Node version from `.nvmrc` (falls back to `lts/*`), runs `npm ci` and `npm run build`.
4. Rsyncs `web/`, `pantheon.yml`, and `wp-cli.yml` to the Pantheon local clone, excluding paths listed in `exclusions.txt` (in the calling repo root) and the theme `src/` directory.
5. Commits and pushes to the Pantheon branch.
6. Optionally promotes to Pantheon Test via `terminus env:deploy`.
7. Posts a Slack failure notification on any job failure.

## Calling Repo Requirements

- An `exclusions.txt` file in the repo root listing any additional rsync exclusions.
- Composer dependencies in the repo root (a `composer.json`).
- A theme `src/` directory with `package.json` (and optionally `.nvmrc`) if `build_frontend` is `true`.
