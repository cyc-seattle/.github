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

Every merge to `main` publishes a new patch version of the repo's npm package.

The repo must use pnpm and hold one package. Its `package.json` holds `"version": "0.0.0-development"`. The git tags and npm hold the real version.

Add `.github/workflows/release.yml` to the repo:

```yaml
name: release

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      bump:
        description: The version part to bump
        type: choice
        options: [patch, minor, major]
        default: patch

permissions:
  contents: write
  id-token: write

jobs:
  release:
    uses: cyc-seattle/.github/.github/workflows/release.yml@v2.0.0
    with:
      bump: ${{ inputs.bump }}
```

The caller must grant both permissions:

| Permission        | Used to                                                  |
| ----------------- | -------------------------------------------------------- |
| `contents: write` | Read the tags, and create the tag and the GitHub release |
| `id-token: write` | Publish to npm with trusted publishing                   |

### How a release is cut

1. [`mathieudutour/github-tag-action`](https://github.com/mathieudutour/github-tag-action) finds the latest `vX.Y.Z` tag and computes the next version. It creates no tag yet.
2. The workflow sets that version in the CI workspace. It does not commit it.
3. The workflow publishes with `pnpm publish`. No npm token is used.
4. If the publish succeeds, the workflow tags the commit `vX.Y.Z` and creates a GitHub release with generated notes.

### Release a minor or major version

1. Merge the change. The merge releases a patch.
2. In the repo's **Actions** tab, run the `release` workflow on `main`. Set **bump** to `minor` or `major`.

Do not start a commit subject with a conventional-commit prefix. The tag action reads `feat:` as a minor bump and `feat!:` as a major bump.

### Set up a new package

1. Publish the first version by hand. npm trusted publishing needs a package that already exists.
2. Tag the commit you published from with that version, for example `v0.1.0`, and push the tag. The tag action counts from it.
3. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `release.yml`. Do not name this repo.

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
