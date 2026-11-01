# .github

Shared files for every cyc-seattle repo.

This repo holds:

- Default community health files, such as the contributing guide and security policy.
- The organization profile.
- Reusable workflows: `ci.yml`, `release.yml`, and `deploy.yml`.
- The shared Renovate preset.

## Run CI from a repo

Add two workflows to the repo. `.github/workflows/pull-request.yml` runs CI:

```yaml
name: pull-request

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v2.0.0
```

`.github/workflows/pr-title.yml` checks the PR title:

```yaml
name: pr-title

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

permissions:
  pull-requests: read

jobs:
  pr-title:
    uses: cyc-seattle/.github/.github/workflows/pr-title.yml@v2.0.0
```

- `ci` installs the tools from `mise.lock`, then runs `mise run install` and `mise run ci`. The install gets the run's `GITHUB_TOKEN`, so Pulumi's plugin downloads are authenticated. Its status check is `ci / ci`.
- `pr-title` checks that the PR title is a conventional commit, with [`amannn/action-semantic-pull-request`](https://github.com/amannn/action-semantic-pull-request). It runs only on pull requests. Its status check is `pr-title / pr-title`.

Only `pr-title.yml` listens for `edited`, so editing a PR's title or description reruns the title check and nothing else. Keep the job IDs `ci` and `pr-title`, because they name the status checks. Require both checks in the repo's ruleset. A repo with a merge queue also runs both on `merge_group`, so the queue sees the same checks.

## Release an npm package from a repo

Every merge to `main` runs [semantic-release](https://github.com/semantic-release/semantic-release) once CI passes. It reads the conventional commits since the last `vX.Y.Z` tag and picks the bump:

| Commit            | Release |
| ----------------- | ------- |
| `fix:`            | Patch   |
| `feat:`           | Minor   |
| `feat!:`, `fix!:` | Major   |
| Any other type    | None    |

Then it publishes to npm with trusted publishing, tags the commit, and creates a GitHub release. It does not comment on issues or pull requests, and it adds no labels.

The repo must hold one package. Its `package.json` holds `"version": "0.0.0-development"`. The git tags and npm hold the real version, and nothing is committed back.

Add a `release` job to the repo's `.github/workflows/pull-request.yml`, after `ci`:

```yaml
jobs:
  ci:
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v2.0.0
  # Releases only from main, and only once CI passes.
  release:
    needs: ci
    if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'
    uses: cyc-seattle/.github/.github/workflows/release.yml@v2.0.0
    permissions:
      contents: write
      id-token: write
```

The job lives in the same workflow as CI so it can wait on it with `needs`. A failed CI skips the release. A repo with a merge queue (see [below](#deploy-to-a-shared-dev-from-a-merge-queue)) has already run CI and its dev deploy on the commit in the queue, so its release runs on `main`'s push with no `needs`. To retry a release that failed before it tagged, run `pull-request` on `main` from the Actions tab: `workflow_dispatch` runs CI again, then the release.

The job must grant both permissions. The rest of the workflow keeps `contents: read`.

| Permission        | Used to                                     |
| ----------------- | ------------------------------------------- |
| `contents: write` | Push the tag, and create the GitHub release |
| `id-token: write` | Publish to npm with trusted publishing      |

semantic-release and its four default plugins run at exact versions, set in `release.yml`. Renovate bumps them in this repo.

### Publish a container image

Pass `image` to also publish a container image when a run releases a new version. The run builds `linux/amd64` and pushes `<image>:<version>` and `<image>:latest` to GHCR. The version has no `v`. A run that releases nothing pushes nothing. The release runs first, so a failed build leaves the tag in place. The image is built inside `release.yml` because a tag pushed with `GITHUB_TOKEN` starts no other workflow.

```yaml
release:
  needs: ci
  if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'
  uses: cyc-seattle/.github/.github/workflows/release.yml@v2.2.0
  permissions:
    contents: write
    id-token: write
    packages: write
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
3. On npmjs.com, add a trusted publisher for the package. Name the repo and the caller's workflow file, `pull-request.yml`. Do not name this repo.

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

| Input         | Does                                                                                |
| ------------- | ----------------------------------------------------------------------------------- |
| `environment` | Required. The GitHub environment the job runs in, for example `dev`                 |
| `task`        | Required. `preview`, `deploy`, `reset`, or `none`. The job fails on any other value |
| `projects`    | The Pulumi projects to run on, separated by spaces. Defaults to empty, for all      |
| `integration` | Runs `mise run integration` after the task. Defaults to false                       |

`none` runs no task, but still checks out, authenticates, and installs, so `integration: true` runs the integration tests alone against what is already deployed. The task and the integration tests get `projects` as `DEPLOY_PROJECTS`. The caller's tasks pick what to run from it, and treat an empty value as their full list.

The caller must grant `id-token: write` and `contents: read`. The job passes the run's `GITHUB_TOKEN` to every step, so Pulumi's plugin downloads from GitHub releases are authenticated and avoid the anonymous rate limit. It caches `~/.pulumi/plugins` between runs, keyed on `pnpm-lock.yaml` and `mise.lock`. The job uses the environment's own approval rules and branch rules, so a `prod` environment that requires a reviewer pauses the job until someone approves.

The job runs one at a time for each repo and environment, and never cancels one in progress. A third deploy to the same environment replaces a second that is still waiting. Its check reads `<caller job> / deploy to <environment>`, for example `dev / deploy to dev`.

### Deploy to a shared dev from a merge queue

When several branches deploy to one dev, a newer deploy replaces an older one that is still waiting, and PRs get cancelled. So deploy to dev only from the repo's merge queue, which tests one PR at a time, and not from a PR's push (cyc-seattle/platform#69). The queue tests the exact commit that lands on `main`, so `main`'s push goes straight to prod:

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
  merge_group:
  push:
    branches: [main]

jobs:
  ci:
    if: github.event_name != 'push'
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v3.0.0
  dev:
    needs: ci
    if: github.event_name == 'merge_group'
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v3.0.0
    permissions:
      id-token: write
      contents: read
    with:
      environment: dev
      task: deploy
      integration: true
  # The check the queue requires. A skipped reusable workflow reports under another name, so this
  # job always runs: on a PR it passes, and in the queue it passes only if dev did.
  dev-passed:
    needs: dev
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: '[[ "$EVENT" != merge_group || "$RESULT" == success ]]'
        shell: bash
        env:
          EVENT: ${{ github.event_name }}
          RESULT: ${{ needs.dev.result }}
  prod-preview:
    if: github.event_name == 'push'
    # ...
```

Then add `merge_group` to the PR title workflow's triggers too. The `pr-title` job skips itself there, and a skipped job passes. In the repo's ruleset, require `ci / ci`, `pr-title / pr-title`, and `dev-passed`, and turn on the merge queue with one entry built at a time. The queue waits only for required checks, so without `dev-passed` it would merge once CI passed.

The `dev` environment must allow the queue's branches, `gh-readonly-queue/main/...`, so leave its branch rule off. The OIDC subject names the environment, not the branch, so Workload Identity needs no change.

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

### Skip a deploy that changes nothing

For `task: preview`, the workflow has an output, `changes`: `'true'` if the preview shows changes, and `'false'` if not. For any other task it is empty. Gate the deploy job on it, so a deploy that would change nothing never starts, and never asks for the environment's approval:

```yaml
prod-preview:
  uses: cyc-seattle/.github/.github/workflows/deploy.yml@v2.4.0
  with:
    environment: prod-preview
    task: preview
prod:
  needs: prod-preview
  if: needs.prod-preview.outputs.changes == 'true'
  uses: cyc-seattle/.github/.github/workflows/deploy.yml@v2.4.0
  with:
    environment: prod
    task: deploy
```

The caller's `preview` task reports the changes. The workflow sets `DEPLOY_CHANGES_FILE` to the path of an empty file, and the task appends one line for each Pulumi project it previews: `true` if that project has changes, `false` if not. Then `changes` is:

| The file holds                    | `changes`     |
| --------------------------------- | ------------- |
| At least one `true`               | `'true'`      |
| Only `false` lines                | `'false'`     |
| Nothing                           | `'true'`      |
| Any line other than true or false | The job fails |

A task that reports nothing keeps deploying every time, so a caller that hasn't adopted this needs no change. A preview that fails fails the job, so it never reads as no changes. To compute a project's line, preview it with `--expect-no-changes`. It exits 0 when nothing changes. When something does, it fails with `no changes were expected but changes were proposed`. Any other failure is a failed preview, so fail the task:

```bash
status=0
pulumi preview --cwd "stacks/$project" --expect-no-changes 2>&1 | tee "$out" || status=$?
if ((status == 0)); then
  echo false >> "$DEPLOY_CHANGES_FILE"
elif grep -q 'no changes were expected.* but changes were proposed' "$out"; then
  echo true >> "$DEPLOY_CHANGES_FILE"
else
  exit "$status"
fi
```

cyc-seattle/platform's `scripts/each-stack` does this for each project.

## Update dependencies with Renovate

Every repo's `renovate.json` extends the shared preset, `renovate-config.json`:

```json
{ "extends": ["local>cyc-seattle/.github:renovate-config"] }
```

The preset decides what Renovate opens and what merges itself. Times are in `America/Los_Angeles`.

| Update                        | PR                                                       | Merges                                |
| ----------------------------- | -------------------------------------------------------- | ------------------------------------- |
| Minor, patch, pin, and digest | One grouped PR per repo, opened before 6am on Mondays    | Itself, once the required checks pass |
| Major                         | One PR each, once a maintainer ticks it on the dashboard | By hand                               |
| Lockfile maintenance          | One PR, opened before 6am on the first of the month      | Itself, once the required checks pass |

- Automerge uses GitHub's native auto-merge, so the repo must allow auto-merge and its ruleset must require the checks. Renovate merges nothing whose checks fail.
- Majors wait on the Dependency Dashboard issue. Tick one there to open its PR.
- PR titles are conventional commits. A runtime dependency (`dependencies` in `package.json`) gets `fix(deps):`, so semantic-release publishes a patch. Anything else gets `chore(deps):` and releases nothing. A grouped PR takes `fix` if any update in it does.
- Actions are pinned to a commit SHA, with the version in a comment, such as `actions/checkout@<sha> # v7.0.1`. A tag can be moved to other code; a SHA can't. Renovate pins any action that isn't pinned yet, and its digest updates keep the SHA and the comment current. The reusable workflows in this repo stay on their tags, because the Workload Identity provider accepts `deploy.yml` only at a tag.
- Node stays at 22: Renovate offers no later `node`, `@types/node`, or `@tsconfig/nodeNN`, and resolves lockfiles with Node 22. ESLint stays at 9.

Check a change to the preset with `renovate-config-validator`, from a directory that holds a copy of it named `renovate.json`:

```sh
npx --package renovate -- renovate-config-validator --strict
```

## Release a workflow change

1. Merge the change to `main`.
2. Create an immutable tag `vX.Y.Z` that follows semantic versioning.

Renovate bumps the tag in each caller. Community files and the Renovate preset come from the default branch and need no tag.
