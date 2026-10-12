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

Every merge to `main` publishes a new patch version of the repo's npm package. To publish a bigger bump, add the `minor` or `major` label to the PR before you merge it.

The repo must use pnpm and hold one package. Its `package.json` holds `"version": "0.0.0-development"`. npm and the git tags hold the real version.

Add `.github/workflows/release.yml` to the repo:

```yaml
name: release

on:
  push:
    branches: [main]

permissions:
  contents: write
  id-token: write
  pull-requests: read

jobs:
  release:
    uses: cyc-seattle/.github/.github/workflows/release.yml@v2.0.0
```

The caller must grant all three permissions:

| Permission            | Used to                                |
| --------------------- | -------------------------------------- |
| `contents: write`     | Create the tag and the GitHub release  |
| `id-token: write`     | Publish to npm with trusted publishing |
| `pull-requests: read` | Read the labels on the merged PR       |

Then set up the package:

1. Publish the first version by hand. npm trusted publishing needs a package that already exists.
2. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `release.yml`. Do not name this repo.

On each run, the workflow:

1. Reads the latest version from npm, and bumps it.
2. Sets the new version in the CI workspace. It does not commit it.
3. Publishes with `pnpm publish`. No npm token is used.
4. Tags the commit `vX.Y.Z`, and creates a GitHub release with generated notes.

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
