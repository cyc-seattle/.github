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

Only `pr-title.yml` listens for `edited`, so editing a PR's title or description reruns the title check and nothing else. Keep the job IDs `ci` and `pr-title`, because they name the status checks. Require both checks in the repo's ruleset.

## Release an npm package from a repo

Every merge to `main` runs [semantic-release](https://github.com/semantic-release/semantic-release) once CI passes. It reads the conventional commits since the last `vX.Y.Z` tag and picks the bump:

| Commit                              | Release |
| ----------------------------------- | ------- |
| `fix:`                              | Patch   |
| `feat:`                             | Minor   |
| Any type with `!`, such as `feat!:` | Major   |
| Any other type                      | None    |

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

The job lives in the same workflow as CI so it can wait on it with `needs`. A failed CI skips the release. To retry a release that failed before it tagged, run `pull-request` on `main` from the Actions tab: `workflow_dispatch` runs CI again, then the release.

The job must grant both permissions. The rest of the workflow keeps `contents: read`.

| Permission        | Used to                                     |
| ----------------- | ------------------------------------------- |
| `contents: write` | Push the tag, and create the GitHub release |
| `id-token: write` | Publish to npm with trusted publishing      |

semantic-release, its four default plugins, and the `conventionalcommits` preset run at exact versions, set in `release.yml`. Renovate bumps them in this repo.

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

| Input                 | Does                                                                                |
| --------------------- | ----------------------------------------------------------------------------------- |
| `environment`         | Required. The GitHub environment the job runs in, for example `dev`                 |
| `task`                | Required. `preview`, `deploy`, `reset`, or `none`. The job fails on any other value |
| `projects`            | The Pulumi projects to run on, separated by spaces. Defaults to empty, for all      |
| `integration`         | Runs `mise run integration` after the task. Defaults to false                       |
| `integration-timeout` | Minutes the integration tests may run before the step fails. Defaults to 60         |

When the integration tests fail, the job uploads the caller's `test-results/` as the `test-results-<environment>` artifact, kept 7 days, so a failed journey's page snapshot and trace can be read. A repo whose tests write elsewhere uploads nothing.

`none` runs no task, but still checks out, authenticates, and installs, so `integration: true` runs the integration tests alone against what is already deployed. On dev it lets deploys go first, and tests the commit last deployed there (see [below](#test-dev-after-each-deploy)). The task and the integration tests get `projects` as `DEPLOY_PROJECTS`. The caller's tasks pick what to run from it, and treat an empty value as their full list.

The caller must grant `id-token: write` and `contents: read`. The job passes the run's `GITHUB_TOKEN` to every step, so Pulumi's plugin downloads from GitHub releases are authenticated and avoid the anonymous rate limit. It caches `~/.pulumi/plugins` between runs, keyed on `pnpm-lock.yaml` and `mise.lock`. The job uses the environment's own approval rules and branch rules, so a `prod` environment that requires a reviewer pauses the job until someone approves.

The job runs one at a time for each repo and environment, and never cancels one in progress. Deploys to `dev` wait in line. Test runs (`task: none`) have a group of their own. For any other environment, a third deploy replaces a second that is still waiting. Its check reads `<caller job> / deploy to <environment>`, for example `dev / deploy to dev`.

Every repo deploys to the same dev, and a concurrency group works only inside one repo. So a `dev` run waits for dev's lock, a ConfigMap named `dev-lock` in the `dev` namespace of the `cyc` cluster, before its task, and holds it through the integration tests. A test run lets any waiting deploy go first. It names the run that holds it. A run that dies without releasing it leaves it to expire after 75 minutes, and the next run takes it over. To free it sooner, delete the ConfigMap once that run has stopped. The service account needs `container.clusters.get` and write access to ConfigMaps in that namespace.

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

### Test dev after each deploy

`main` deploys to dev on each push, then previews and deploys prod, and a separate workflow runs the integration tests against dev after each deploy. A test run takes 12–18 minutes, so it stays out of the deploy's way: deploys go ahead of a waiting test run, and prod never waits behind one. Merges during a run collapse into one run of the newest `main`. Only `main` deploys to dev, so leave the `dev` environment's branch rule on `main`.

`.github/workflows/pull-request.yml`:

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]
  # Retries main's deploys.
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    if: github.event_name == 'pull_request'
    uses: cyc-seattle/.github/.github/workflows/ci.yml@v4.0.0
  dev:
    if: github.ref == 'refs/heads/main'
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v4.0.0
    permissions:
      id-token: write
      contents: read
    with:
      environment: dev
      task: deploy
  # A run started with GITHUB_TOKEN may start only a workflow_dispatch, so the tests run in a
  # workflow of their own, which keeps them out of this run.
  integration:
    needs: dev
    runs-on: ubuntu-latest
    permissions:
      actions: write
    steps:
      - run: gh workflow run integration.yml -R "$GITHUB_REPOSITORY" --ref main
        env:
          GH_TOKEN: ${{ github.token }}
  prod-preview:
    needs: dev
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v4.0.0
    # ...
  prod:
    needs: prod-preview
    if: needs.prod-preview.outputs.changes == 'true'
    # ...
```

`.github/workflows/integration.yml`:

```yaml
name: integration

on:
  workflow_dispatch:

# One run at a time, and at most one waiting. A newer run replaces one that's waiting.
concurrency:
  group: integration
  cancel-in-progress: false

permissions:
  contents: read

jobs:
  dev:
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v4.0.0
    permissions:
      id-token: write
      contents: read
    with:
      environment: dev
      task: none
      integration: true
  report:
    needs: dev
    if: always() && needs.dev.outputs.skipped != 'true'
    uses: cyc-seattle/.github/.github/workflows/nightly-report.yml@v4.0.0
    permissions:
      actions: read
      contents: read
      issues: write
    with:
      result: ${{ needs.dev.result }}
      assignee: <maintainer>
      title: Integration tests failed on dev
      tested: ${{ needs.dev.outputs.tested }}
      passed: ${{ needs.dev.outputs.passed }}
```

On dev, `task: none` changes how the run takes dev's lock and what it tests:

- **Deploys go first.** A deploy run waiting for the lock keeps a key in the `dev-lock-deploys` ConfigMap, and refreshes it every 30 seconds with a 5-minute expiry. A test run takes the lock only while no deploy is waiting, and if one started waiting as it took it, it lets go. A running test is never stopped.
- **It tests what's on dev.** Each dev deploy that succeeds records its commit as `deployed` in this repo's `dev-state-<owner>.<repo>` ConfigMap. Once a test run holds the lock, it checks out that commit, which may be newer than its own if a deploy went first, and installs again if it changed.
- **It runs only when something new is on dev.** Each integration run on dev, including the nightly, records the commit it tested as `tested`, and as `passed` if the tests passed. A test run skips the tests when `deployed` is `tested`, and sets `skipped` to `true`. A re-run of the job runs them anyway.

| Output    | Holds, for `task: none` on dev                                               |
| --------- | ---------------------------------------------------------------------------- |
| `tested`  | The commit the tests ran on. Empty if they didn't run                        |
| `passed`  | The last commit the tests passed on, before this run. Empty if none recorded |
| `skipped` | `true` if nothing was deployed since the tests last ran                      |

The report keeps its own issue, apart from the nightly one, because the nightly tests every repo together. Given `tested` and `passed`, a failure's comment lists the commits merged between them, each with its PR's number, and links their diff. It needs `contents: read` for that.

A test run has its own concurrency group, `integration-<repo>-<environment>`, apart from the repo's deploys, so it never holds up the repo's next deploy before dev's lock can put that deploy first.

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

### Test dev nightly

The tests after each deploy run only when something new of the repo's is on dev. Test dev once a night too, as a
backstop for what changes there between deploys, such as another repo's. Deploy `main` to dev with the integration
tests, then report the result with `nightly-report.yml`. It keeps one open issue per repo: a failed night opens it,
assigned to `assignee` so GitHub emails them, or comments on it, with the failing scenarios and the run's link. A
green night closes it. The dev run takes dev's lock like any other.

```yaml
on:
  schedule:
    - cron: "17 10 * * *"
  workflow_dispatch:

jobs:
  dev:
    uses: cyc-seattle/.github/.github/workflows/deploy.yml@v4.0.0
    permissions:
      id-token: write
      contents: read
    with:
      environment: dev
      task: deploy
      integration: true
  report:
    needs: dev
    if: always()
    uses: cyc-seattle/.github/.github/workflows/nightly-report.yml@v4.0.0
    permissions:
      actions: read
      issues: write
    with:
      result: ${{ needs.dev.result }}
      assignee: <maintainer>
```

It finds the failing scenarios in the failed jobs' logs, from vitest's `FAIL` lines and Playwright's numbered ones.
The issue's title defaults to `Nightly integration run failed`, and is how the workflow finds it again. The report takes the caller's permissions.

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
