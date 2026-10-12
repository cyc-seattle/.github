# .github

Shared files for every cyc-seattle repo.

This repo holds:

- Default community health files, such as the contributing guide and security policy.
- The organization profile.
- Reusable workflows: `ci.yml` and `release.yml`.
- The shared Renovate preset.

## Run CI from a repo

Add `.github/workflows/pull-request.yml` to the repo:

```yaml
name: pull-request

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v1.0.0
```

The workflow installs the tools from `mise.lock`, then runs `mise run install` and `mise run ci`.
The required status check is `ci / ci`.

## Release an npm package from a repo

The repo must use pnpm and changesets, and its CI caller must be `pull-request.yml` with `workflow_dispatch`.

Add `.github/workflows/release.yml` to the repo:

```yaml
name: release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  id-token: write
  actions: write

jobs:
  release:
    uses: cyc-seattle/.github/.github/workflows/release.yml@v1.0.0
```

The caller must grant all four permissions:

| Permission             | Used to                                                   |
| ---------------------- | --------------------------------------------------------- |
| `contents: write`      | Push the version commit, tags, and releases               |
| `pull-requests: write` | Open the "Version Packages" PR                            |
| `id-token: write`      | Publish to npm with trusted publishing                    |
| `actions: write`       | Start `pull-request.yml` on the "Version Packages" branch |

Then set up the repo:

1. In the repo settings, under **Actions > General**, turn on **Allow GitHub Actions to create and approve pull requests**.
2. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `release.yml`. Do not name this repo.

The workflow runs `changesets/action`. If there are changesets, it opens or updates the "Version Packages" PR, then starts CI on it. When that PR merges, the workflow publishes with `pnpm exec changeset publish`. No npm token is used.

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
