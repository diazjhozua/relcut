# relcut

[![CI](https://github.com/diazjhozua/relcut/actions/workflows/ci.yml/badge.svg)](https://github.com/diazjhozua/relcut/actions/workflows/ci.yml)
[![Latest release](https://img.shields.io/github/v/release/diazjhozua/relcut)](https://github.com/diazjhozua/relcut/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Cut releases automatically. A reusable GitHub Actions workflow that turns
Conventional Commits on `main` into SemVer versions, changelogs, and GitHub
Releases — powered by [release-please](https://github.com/googleapis/release-please).

## How it works

1. You merge commits to `main` using [Conventional Commits](https://www.conventionalcommits.org)
   (`feat:`, `fix:`, `feat!:` ...).
2. relcut opens (and keeps updating) a **Release PR** containing the next
   version bump and generated `CHANGELOG.md`.
3. When you merge that Release PR, relcut creates the git tag and the GitHub
   Release with the release notes — and moves the major tag (e.g. `v1`) to it.

| Commit | Version bump |
|---|---|
| `fix:`, `perf:` | patch (1.2.3 → 1.2.4) |
| `feat:` | minor (1.2.3 → 1.3.0) |
| `feat!:` / `BREAKING CHANGE:` footer | major (1.2.3 → 2.0.0) |
| `chore:`, `docs:`, `test:`, `ci:` | none |

## Usage

Add `.github/workflows/release.yml` to your repo:

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    uses: diazjhozua/relcut/.github/workflows/release.yml@v1
    with:
      release-type: simple # or node, python, go, java, rust, ...
```

That's it. Merge conventional commits to `main` and merge the Release PR when
you're ready to ship.

### Inputs

| Input | Default | Description |
|---|---|---|
| `release-type` | `simple` | release-please strategy; controls which version files get bumped (`simple` = `version.txt`, `node` = `package.json`, `python` = `pyproject.toml`, etc.) |
| `target-branch` | `main` | Branch releases are cut from |
| `update-major-tag` | `true` | Keep a moving major tag (`v1`, `v2`, ...) pointing at the latest release. Also applies while you are pre-1.0 (moves a `v0` tag) |
| `config-file` | `release-please-config.json` | Path to a release-please config file (custom changelog sections, monorepos) |
| `manifest-file` | `.release-please-manifest.json` | Path to the release-please manifest tracking released versions |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `release-token` | no | Token used to create the release PR, tags, and releases. Defaults to `GITHUB_TOKEN`. |

> **Chaining workflows off releases?** Tags and releases created with the
> default `GITHUB_TOKEN` do **not** trigger other workflows — so
> `on: push: tags` or `on: release` in your repo will silently never fire.
> Pass a PAT (or GitHub App token) with `contents: write` +
> `pull-requests: write` to fix that:
>
> ```yaml
> jobs:
>   release:
>     uses: diazjhozua/relcut/.github/workflows/release.yml@v1
>     secrets:
>       release-token: ${{ secrets.RELEASE_PAT }}
> ```

### Outputs

| Output | Description |
|---|---|
| `release_created` | `"true"` when a release was created in this run |
| `tag_name` | Tag of the created release, e.g. `v1.2.3` |
| `version` | Version of the created release, e.g. `1.2.3` |

Use outputs to chain jobs (e.g. publish artifacts only when a release happened):

```yaml
jobs:
  release:
    uses: diazjhozua/relcut/.github/workflows/release.yml@v1

  publish:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Shipping ${{ needs.release.outputs.tag_name }}"
```

## Enforcing conventional PR titles

The whole system degrades silently if commits aren't conventional — no error,
just "no release ever happens". Add the companion linter so PRs fail fast at
review time instead:

```yaml
# .github/workflows/lint-pr.yml
name: Lint PR

on:
  pull_request:
    types: [opened, edited, synchronize]

permissions:
  pull-requests: read

jobs:
  lint:
    uses: diazjhozua/relcut/.github/workflows/lint-pr.yml@v1
```

## Requirements

- **Allow Actions to create pull requests** — by default GitHub blocks this,
  and the run fails with
  `GitHub Actions is not permitted to create or approve pull requests`.
  Enable it in the caller repo:
  **Settings → Actions → General → Workflow permissions** → check
  *"Allow GitHub Actions to create and approve pull requests"* — or via CLI:

  ```bash
  gh api -X PUT repos/<owner>/<repo>/actions/permissions/workflow \
    -f default_workflow_permissions=write \
    -F can_approve_pull_request_reviews=true
  ```

- Squash-merge PRs and make the PR title a conventional commit (or enforce
  conventional commits on every commit).
- The caller repo must grant `contents: write` and `pull-requests: write`
  permissions (as in the snippet above).
- If your org restricts Actions, allow `googleapis/release-please-action`.

## Fine-tuning

Add a `release-please-config.json` / `.release-please-manifest.json` to the
caller repo for advanced control (changelog sections, monorepo packages,
initial versions). See the
[release-please docs](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md).

## Versioning policy

relcut follows SemVer, and the workflow interface is the API:

- **Pin `@v1`** (recommended). The `v1` tag always points at the latest
  `v1.x.y` release, so you get fixes and new inputs automatically.
- Within `v1`: existing inputs, outputs, secrets, and their defaults will not
  change behavior or be removed. New inputs may be added (always optional,
  with backward-compatible defaults).
- A breaking change — removing/renaming an input or output, changing a
  default, requiring a new permission — ships as `v2` with a migration note
  in the release notes. `v1` keeps working and keeps receiving critical fixes
  until noted otherwise.
- Pinning `@main` gets you unreleased commits and future majors without
  warning; only do that if you accept breakage.

## Dogfooding

This repository releases itself with its own workflow — see
[`.github/workflows/self-release.yml`](.github/workflows/self-release.yml).

## License

[MIT](LICENSE)
