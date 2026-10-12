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

Every merge to `main` runs [semantic-release](https://github.com/semantic-release/semantic-release). It reads the conventional commits since the last `vX.Y.Z` tag and picks the bump:

| Commit            | Release |
| ----------------- | ------- |
| `fix:`            | Patch   |
| `feat:`           | Minor   |
| `feat!:`, `fix!:` | Major   |
| Any other type    | None    |

Then it publishes to npm with trusted publishing, tags the commit, and creates a GitHub release. It does not comment on issues or pull requests, and it adds no labels.

The repo must hold one package. Its `package.json` holds `"version": "0.0.0-development"`. The git tags and npm hold the real version, and nothing is committed back.

Add `.github/workflows/release.yml` to the repo:

```yaml
name: release

on:
  push:
    branches: [main]

permissions:
  contents: write
  id-token: write

jobs:
  release:
    uses: cyc-seattle/.github/.github/workflows/release.yml@v2.0.0
```

The caller must grant both permissions:

| Permission        | Used to                                     |
| ----------------- | ------------------------------------------- |
| `contents: write` | Push the tag, and create the GitHub release |
| `id-token: write` | Publish to npm with trusted publishing      |

### Set up a new package

1. Publish the first version by hand, for example `0.1.0`. npm trusted publishing needs a package that already exists.
2. Tag the commit you published from with that version, for example `v0.1.0`, and push the tag. semantic-release counts from it.
3. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `release.yml`. Do not name this repo.

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
