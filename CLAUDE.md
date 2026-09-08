# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository contains a reusable GitHub Actions workflow (`.github/workflows/deploy.yml`) that downstream WordPress repositories call via `workflow_call` to deploy to [Pantheon](https://pantheon.io/). The other two workflows here support it rather than deploy anything: `lint.yml` (actionlint + shellcheck on every workflow) and `release-tag.yml` (moves the floating major tag on release).

## Editing deploy.yml

Every downstream site calls this one file, so a mistake ships to all of them at once and surfaces mid-deploy. Two consequences:

- **Lint before pushing.** CI runs `actionlint` (which also runs shellcheck over `run:` blocks and validates which contexts are legal per key). Both binaries are pinned in `lint.yml` — actionlint verified against the publisher's checksums file, shellcheck against a digest recorded inline because it publishes none. To reproduce CI locally, install **the same versions `lint.yml` pins** and run `actionlint -shellcheck <path-to-pinned-shellcheck>` from the repo root with no path argument.
  - Version skew here is not theoretical: shellcheck 0.11.0 stopped flagging `cmd || true` as SC2015, so a newer local shellcheck passes code that an older runner image rejects. That is why CI pins rather than using the image's copy — and why a local run with a mismatched shellcheck proves little. When bumping either version, bump it in `lint.yml` and update the recorded digest.
- **Version deliberately.** See Versioning below — a breaking change needs a major bump, not a merge to `main`.

## Versioning

Releases are tagged `vX.Y.Z`; `release-tag.yml` moves the `vX` alias to the newest release in that line whenever a non-prerelease is published. Consumers should pin `@v1`.

Breaking (major): removing or renaming an input, making an optional input required, requiring a new secret or repo variable, or changing a default so an existing caller's deploy behaves differently. Everything else — new optional inputs, internal restructuring, pinned action bumps — is minor or patch.

To cut a release, publish a GitHub Release with a `vX.Y.Z` tag; the alias moves on its own. `release-tag.yml` force-pushes only a bare `vN` alias, guarded by an exact `^v[0-9]+\.[0-9]+\.[0-9]+$` match on the release tag, so it cannot overwrite a semver tag.

## How Downstream Repos Use This

Consuming repos reference this workflow with:

```yaml
uses: Taupecat-Studios-Client-Sites/.github/.github/workflows/deploy.yml@v1
```

The doubled `.github` is correct: this repo is named `.github`, and the workflow sits at `.github/workflows/deploy.yml` within it. Consumers pin a tag (`@v1`), not `@main` — see Versioning below.

Required inputs:
- `pantheon_branch` — the Pantheon Git branch to deploy to (e.g. `master`, `dev`)

Optional inputs:
- `deploy_to_test` (bool, default `false`) — promotes to Pantheon Test after pushing to Dev, but only if the push actually happened
- `build_frontend` (bool, default `true`) — whether to run the Node/npm build step
- `web_docroot` (bool, default `true`) — whether the site uses a `web/` subdirectory as the Pantheon document root; when `false`, the contents of `web/` are rsynced directly into the Pantheon repo root instead
- `src_dir` (string) — explicit path to the directory containing `package.json` and optionally `.nvmrc`; defaults to `web/wp-content/themes/<THEME_DIR>/src`
- `extra_exclusions` (string, one path per line) — additional rsync exclusions layered on top of the built-in defaults (`.DS_Store`, `wp-content/uploads`)
- `legacy_peer_deps` (bool, default `false`) — passes `--legacy-peer-deps` to `npm ci`, for sites with unresolved peer dependency conflicts
- `git_user_name` / `git_user_email` (string) — author identity for the commit pushed to Pantheon; defaults to the GitHub Actions bot

Required secrets in the calling repo: `PANTHEON_SSH_KEY`, `PANTHEON_MACHINE_TOKEN`, `AUTH_JSON`, `ENV`, `ACTION_MONITORING_SLACK`.

Required variable in the calling repo: `PANTHEON_SITE_NAME`. `THEME_DIR` is required unless `src_dir` is passed or `build_frontend` is false (it's also used to derive the theme `src/` rsync exclusion when `THEME_DIR` is set).

## Deployment Flow

1. Clones the Pantheon Git repo (via Terminus `local:clone`) in the background while the GitHub repo checks out in parallel.
2. Writes `auth.json`/`.env` from secrets, conditionally installs Subversion (only if `composer.json`/`composer.lock` reference an `"type": "svn"` source), then runs `composer install --no-dev`.
3. If `build_frontend` is true: resolves the `src_dir`, detects the Node version from `.nvmrc` (falls back to `lts/*`), runs `npm ci` (optionally with `--legacy-peer-deps`) and `npm run build`.
4. Waits for the backgrounded Pantheon clone, builds the rsync exclusion list (default exclusions + `extra_exclusions`), then rsyncs `web/`, `pantheon.yml`, and `wp-cli.yml` to the Pantheon local clone — either into a `web/` subdirectory or flattened to the repo root depending on `web_docroot`. The theme `src/` directory is always excluded when `THEME_DIR` is set.
5. Commits and pushes to the Pantheon branch, skipping the push if there's nothing to commit. The commit message is the source commit's subject plus `Source-Repo`/`Source-Ref`/`Source-Commit`/`Source-Run` git trailers.
6. Optionally promotes to Pantheon Test via `terminus env:deploy`, gated on the commit step's `pushed` output. A run whose rsync produced no change has nothing new on Dev to promote — which is every workflow-only commit in the calling repo, since only `web/`, `pantheon.yml` and `wp-cli.yml` are rsynced.
7. Posts a Slack failure notification on any job failure.

## Key Design Notes

- The job is serialized with a `concurrency` group keyed on `PANTHEON_SITE_NAME` + `pantheon_branch`, because Pantheon permits only one `sync_code` workflow per site at a time. `cancel-in-progress` is deliberately `false` — cancelling between the rsync and the push would leave the Pantheon working copy dirty for the next run.
- Provenance travels as git trailers rather than in the commit subject or author field: the Pantheon repo holds a built artifact in its own history, so its SHAs don't exist on GitHub. The trailers survive promotion to Test and Live and are readable with `git log -1 --pretty='%(trailers:key=Source-Commit,valueonly=true)'`. The Pantheon deploy note (`--note`) gets a separate single-line `DEPLOY_NOTE` output instead, since it's a short field.
- Every caller-controlled value — `inputs.*`, `vars.*`, `secrets.*`, and step outputs derived from them — is passed into `run:` blocks via `env:` rather than interpolated into the script body, so a quote or `$` in an input can't become a shell metacharacter. Three interpolations remain inside `run:` bodies on purpose: `github.token` and `runner.temp` are runner-provided, and the `legacy_peer_deps` ternary evaluates to one of two literals. Anything else appearing there is a regression.
  - Consequence: use `"$HOME/..."` rather than `~/...` in those blocks, since `~` doesn't expand inside the quotes this style requires.
- The Terminus session is encrypted with the machine token and cached between runs (`terminus-session.enc`) to avoid re-authenticating on every run.
- The Pantheon clone starts async (backgrounded `&`) so it overlaps with the GitHub checkout and build steps — the workflow waits for it before rsyncing.
- Rsync exclusions are built entirely from inputs (`DEFAULT_EXCLUSIONS` env plus the `extra_exclusions` input) — there is no `exclusions.txt` file expected in the calling repo.
- The theme `src` directory is excluded from the Pantheon rsync only when `THEME_DIR` is set (it's used to build the `--exclude=wp-content/themes/${THEME_DIR}/src` rsync flag), regardless of how `src_dir` was resolved for the build step.
- Subversion is installed on the runner only when `composer.json`/`composer.lock` actually reference an SVN-sourced package, to avoid paying for an unnecessary `apt-get update`.
