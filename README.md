# .github

Shared files for every cyc-seattle repo.

This repo holds:

- Default community health files, such as the contributing guide and security policy.
- The organization profile.
- Reusable workflows. Only `ci.yml` exists so far.
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

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
