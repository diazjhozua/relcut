# relcut

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
    uses: <owner>/relcut/.github/workflows/release.yml@v1
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
| `update-major-tag` | `true` | Keep a moving major tag (`v1`, `v2`, ...) pointing at the latest release |

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
    uses: <owner>/relcut/.github/workflows/release.yml@v1

  publish:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Shipping ${{ needs.release.outputs.tag_name }}"
```

## Requirements

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

## Dogfooding

This repository releases itself with its own workflow — see
[`.github/workflows/self-release.yml`](.github/workflows/self-release.yml).

## License

[MIT](LICENSE)
