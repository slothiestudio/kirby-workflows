# kirby-workflows

Shared GitHub Actions workflows for slothiestudio's Kirby sites. Each site keeps
four short caller files; checks, deploys and the release PR live here once.

| Workflow         | What it does                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `deploy.yml`     | Installs Composer (and optionally npm) dependencies and deploys over SSH with [ssh-deploy](ssh-deploy/). Clears Kirby's cache and the Staticache pages afterwards. |
| `check.yml`      | Runs on pull requests: PHP lint, `npm run format:check`, a smoke test and/or Playwright. Auto-merges Dependabot patch groups and deploys them to preview.          |
| `release-pr.yml` | Keeps one "Release: main → live" PR open with a changelog. Merging it deploys live.                                                                                |

The deploy step itself is the [`ssh-deploy`](ssh-deploy/) action in this repo (rsync over SSH),
usable on its own as `slothiestudio/kirby-workflows/ssh-deploy@v1`.

## Site layout these workflows expect

- Kirby with the web root in `public_html/`, `server.php` for PHP's built-in server
- Accounts, cache and sessions in `storage/`, content in `content/`
- Branches: `main` deploys to preview, `live` deploys to live
- Optional: Staticache in `public_html/static/`, `npm run build`, `npm run format:check`, Playwright in `npm test`

## Setup in a site

**1. GitHub environments** `preview` and `Live` (Settings → Environments), each with:

- secrets `SERVER_URL`, `USERNAME`, and `SSH_KEY` (preferred) or `PASSWORD`; optional `SSH_PASSPHRASE`, `SSH_FINGERPRINT`
- variable `SERVER_DIRECTORY`: the project root on the server (not `public_html`)

The deploy job reads these from the environment directly, so callers don't pass secrets.
This also works for site repos outside the slothiestudio organization.

**2. Settings → Actions → General**: allow GitHub Actions to create pull requests (for the release PR).

**3. Caller workflows** in the site's `.github/workflows/`:

```yaml
# preview.yml
name: PREVIEW Deploy website on push

on:
  push:
    branches: ["main"]
  # Started by the Dependabot auto-merge in check.yml (merges made by
  # workflows don't fire push events).
  workflow_dispatch:

jobs:
  deploy:
    uses: slothiestudio/kirby-workflows/.github/workflows/deploy.yml@v1.6.1
    with:
      environment: preview
      php-version: "8.4"
    # Passed by name: secrets: inherit doesn't work across organizations.
    secrets:
      SERVER_URL: ${{ secrets.SERVER_URL }}
      USERNAME: ${{ secrets.USERNAME }}
      SSH_KEY: ${{ secrets.SSH_KEY }}
      PASSWORD: ${{ secrets.PASSWORD }}
      SSH_PASSPHRASE: ${{ secrets.SSH_PASSPHRASE }}
      SSH_FINGERPRINT: ${{ secrets.SSH_FINGERPRINT }}
```

```yaml
# live.yml
name: LIVE Deploy website on push

on:
  push:
    branches: ["live"]
  # Started by the Dependabot auto-merge of live patches in check.yml.
  workflow_dispatch:

jobs:
  deploy:
    uses: slothiestudio/kirby-workflows/.github/workflows/deploy.yml@v1.6.1
    with:
      environment: Live
      php-version: "8.4"
    # Passed by name: secrets: inherit doesn't work across organizations.
    secrets:
      SERVER_URL: ${{ secrets.SERVER_URL }}
      USERNAME: ${{ secrets.USERNAME }}
      SSH_KEY: ${{ secrets.SSH_KEY }}
      PASSWORD: ${{ secrets.PASSWORD }}
      SSH_PASSPHRASE: ${{ secrets.SSH_PASSPHRASE }}
      SSH_FINGERPRINT: ${{ secrets.SSH_FINGERPRINT }}
```

```yaml
# check.yml
name: CHECK Pull request

on:
  pull_request:
    branches: ["main", "live"]

# A new push to a PR cancels its older, still running check.
concurrency:
  group: check-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  check:
    uses: slothiestudio/kirby-workflows/.github/workflows/check.yml@v1.6.1
    permissions:
      contents: write
      pull-requests: write
      actions: write
    with:
      php-version: "8.4"
```

```yaml
# release-pr.yml
name: RELEASE Keep main → live PR up to date

on:
  push:
    branches: ["main", "live"]
  workflow_dispatch:

# One run at a time, so two quick merges can't open two release PRs.
concurrency:
  group: release-pr
  cancel-in-progress: false

jobs:
  release-pr:
    uses: slothiestudio/kirby-workflows/.github/workflows/release-pr.yml@v1.6.1
    permissions:
      contents: read
      pull-requests: write
```

Keep the file names `preview.yml` and `release-pr.yml`: the Dependabot auto-merge starts them by name.

## Inputs

### `deploy.yml`

| Input          | Default | Description                                                                  |
| -------------- | ------- | ---------------------------------------------------------------------------- |
| `environment`  |         | GitHub environment with the server secrets (required)                        |
| `php-version`  |         | PHP version for `composer install`; match the server (required)              |
| `node-build`   | `false` | Run `npm ci` and `npm run build` before deploying                            |
| `node-version` | `24`    | Node version for the build                                                   |
| `exclude`      |         | Extra paths to keep off the server (and protect from deletion), one per line |

The deploy deletes files on the server that aren't in the repo, except excluded paths.
Always excluded: `content/`, `content-backup-*`, `storage/`, `public_html/media/`, the
Kirby license, `config.localhost.php`, and source/tooling files (`src/`, `assets/`, `bin/`,
`tests/`, `package.json`, `composer.json`, configs, README). Dev dependencies aren't installed.

### `check.yml`

| Input          | Default  | Description                                                         |
| -------------- | -------- | ------------------------------------------------------------------- |
| `php-version`  |          | PHP version (required)                                              |
| `php-lint`     | `true`   | `php -l` on `public_html/`, `site/` (without plugins), `server.php` |
| `node-build`   | `false`  | `npm run build`                                                     |
| `node-version` | `24`     | Node version                                                        |
| `format-check` | `true`   | `npm run format:check`                                              |
| `smoke-urls`   | `/panel` | Paths that must load via `server.php`, one per line; empty skips    |
| `playwright`   | `false`  | `npm test` with Chromium and WebKit; uploads the report on failure  |
| `notify`       |          | GitHub usernames to @mention (and so email) when Dependabot opens a PR, auto-merges a patch, or fails its checks |
| `preview-url`  |          | Preview site URL, linked in that email |
| `live-url`     |          | Live site URL, linked in the email after a live patch auto-merge |

## Versions

Sites pin a release (`@v1.6.1`), so a change here never reaches a site unreviewed.
Dependabot (`package-ecosystem: github-actions`) opens a PR in each site for new releases.

To release: merge to `main`, then tag `vX.Y.Z` and move `v1` (`git tag -f v1 && git push -f origin v1`).
Breaking changes (renamed inputs, new required secrets) get a new major version.
