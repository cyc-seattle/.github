# .github

Shared files for every cyc-seattle repo.

This repo holds:

- Default community health files, such as the contributing guide and security policy.
- The organization profile.
- Reusable workflows: `ci.yml`, `release.yml`, and `deploy.yml`.
- The shared Renovate preset.

## Run CI from a repo

Add `.github/workflows/pull-request.yml` to the repo:

```yaml
name: pull-request

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v2.0.0
  pr-title:
    uses: cyc-seattle/.github/.github/workflows/pr-title.yml@v2.0.0
    permissions:
      pull-requests: read
```

The workflow has two jobs:

- `ci` installs the tools from `mise.lock`, then runs `mise run install` and `mise run ci`. Its status check is `ci / ci`.
- `pr-title` checks that the PR title is a conventional commit, with [`amannn/action-semantic-pull-request`](https://github.com/amannn/action-semantic-pull-request). It runs only on pull requests. Its status check is `pr-title / pr-title`.

The `edited` type runs both jobs again when the PR title changes. Require both checks in the repo's ruleset.

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

### Publish a container image

Pass `image` to also publish a container image when a run releases a new version. The run builds `linux/amd64` and pushes `<image>:<version>` and `<image>:latest` to GHCR. The version has no `v`. A run that releases nothing pushes nothing. The release runs first, so a failed build leaves the tag in place. The image is built inside `release.yml` because a tag pushed with `GITHUB_TOKEN` starts no other workflow.

```yaml
permissions:
  contents: write
  id-token: write
  packages: write

jobs:
  release:
    uses: cyc-seattle/.github/.github/workflows/release.yml@v2.2.0
    with:
      image: ghcr.io/cyc-seattle/clubspot-fake
```

| Input           | Does                                                                |
| --------------- | ------------------------------------------------------------------- |
| `image`         | The image name, with registry. Leave it out to publish no image     |
| `image-context` | The Docker build context. Defaults to `.`                           |
| `image-file`    | The Dockerfile, relative to the repo root. Defaults to `Dockerfile` |

The image is built by a separate `image` job, so only callers that pass `image` grant `packages: write`. Callers that publish no image need no change. The job checks out the repo, which `contents: write` already covers. The image carries the `org.opencontainers.image.source` label, so GHCR links the package to the caller's repo and gives it that repo's visibility. A package pushed for the first time under a private repo is private. Make the repo public before the first release if the image must be public.

### Set up a new package

1. Publish the first version by hand, for example `0.1.0`. npm trusted publishing needs a package that already exists.
2. Tag the commit you published from with that version, for example `v0.1.0`, and push the tag. semantic-release counts from it.
3. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `release.yml`. Do not name this repo.

## Deploy from a repo

`deploy.yml` runs a Pulumi preview or deploy from GitHub Actions, with no stored credentials. It authenticates to Google Cloud with Workload Identity Federation. Call it once for each environment:

```yaml
jobs:
  dev:
    needs: ci
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v2.1.0
    permissions:
      id-token: write
      contents: read
    with:
      environment: dev
      task: deploy
      integration: true
```

| Input         | Does                                                                        |
| ------------- | --------------------------------------------------------------------------- |
| `environment` | Required. The GitHub environment the job runs in, for example `dev`         |
| `task`        | Required. `preview`, `deploy`, or `reset`. The job fails on any other value |
| `integration` | Runs `mise run integration` after the task. Defaults to false               |

The caller must grant `id-token: write` and `contents: read`. The job uses the environment's own approval rules and branch rules, so a `prod` environment that requires a reviewer pauses the job until someone approves.

The job runs one at a time for each repo and environment, and never cancels one in progress. A third deploy to the same environment replaces a second that is still waiting.

The environment must hold these variables:

| Variable                         | Holds                                             |
| -------------------------------- | ------------------------------------------------- |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | The full name of the Workload Identity provider   |
| `GCP_SERVICE_ACCOUNT`            | The email of the service account to impersonate   |
| `PULUMI_BACKEND_URL`             | The GCS state bucket, for example `gs://<bucket>` |

The job sets `PULUMI_STACK` to `dev` for the `dev` environment and `prod` for any other. The repo's `mise.toml` must define these tasks:

| Task          | Does                                                         |
| ------------- | ------------------------------------------------------------ |
| `install`     | Installs the dependencies                                    |
| `preview`     | Previews the stack. The job writes its output to the summary |
| `deploy`      | Applies the stack                                            |
| `reset`       | Only for `task: reset`. Resets the environment               |
| `integration` | Runs the integration tests against the deployed environment  |

A repo that never sets `integration: true` can leave out that task.

The auth step sets `GOOGLE_APPLICATION_CREDENTIALS` to the credentials it creates. The caller's `mise.toml` must not override it when it is already set, or Pulumi loses those credentials. For example:

```toml
[env]
GOOGLE_APPLICATION_CREDENTIALS = "{{ get_env(name='GOOGLE_APPLICATION_CREDENTIALS', default='<path for local deploys>') }}"
```

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
