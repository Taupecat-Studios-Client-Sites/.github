# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository contains a single reusable GitHub Actions workflow (`.github/workflows/deploy.yml`) that downstream WordPress repositories call via `workflow_call` to deploy to [Pantheon](https://pantheon.io/).

## How Downstream Repos Use This

Consuming repos reference this workflow with:

```yaml
uses: <org>/<this-repo>/.github/workflows/deploy.yml@main
```

Required inputs:
- `pantheon_branch` — the Pantheon Git branch to deploy to (e.g. `master`, `dev`)

Optional inputs:
- `deploy_to_test` (bool, default `false`) — promotes to Pantheon Test after pushing to Dev
- `build_frontend` (bool, default `true`) — whether to run the Node/npm build step
- `src_dir` (string) — explicit path to the directory containing `package.json` and optionally `.nvmrc`; defaults to `web/wp-content/themes/<THEME_DIR>/src`

Required secrets in the calling repo: `PANTHEON_SSH_KEY`, `PANTHEON_MACHINE_TOKEN`, `AUTH_JSON`, `ENV`, `ACTION_MONITORING_SLACK`.

Required variable in the calling repo: `PANTHEON_SITE_NAME`. `THEME_DIR` is required unless `src_dir` is passed or `build_frontend` is false.

## Deployment Flow

1. Clones the Pantheon Git repo (via Terminus `local:clone`) in the background while the GitHub repo checks out in parallel.
2. Runs `composer install --no-dev` using `auth.json` and `.env` secrets.
3. If `build_frontend` is true: resolves the `src_dir`, detects the Node version from `.nvmrc` (falls back to `lts/*`), runs `npm ci` and `npm run build`.
4. Rsyncs `web/`, `pantheon.yml`, and `wp-cli.yml` to the Pantheon local clone, excluding paths listed in `exclusions.txt` (expected in the calling repo root) and the theme `src/` directory.
5. Commits and pushes to the Pantheon branch.
6. Optionally promotes to Pantheon Test via `terminus env:deploy`.
7. Posts a Slack failure notification on any job failure.

## Key Design Notes

- The Terminus session is encrypted with the machine token and cached between runs (`terminus-session.enc`) to avoid re-authenticating on every run.
- The Pantheon clone starts async (backgrounded `&`) so it overlaps with the GitHub checkout and build steps — the workflow waits for it before rsyncing.
- File exclusions live in `exclusions.txt` in the **calling** repo root, not in this repo.
- The `src` directory inside the theme is always excluded from the Pantheon rsync regardless of how `src_dir` is resolved.
