# AGENTS.md

Guidance for coding agents working in this repository.

## Overview

The cyc-seattle org's shared GitHub files:

- `.github/workflows/`: the reusable `ci`, `pr-title`, `release` and `deploy` workflows that every repo calls
- `renovate-config.json`: the Renovate preset every repo extends
- `profile/README.md`: the org's profile page
- `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`: the default community files for repos that have none

`README.md` documents each workflow and the preset for the repos that use them. Keep it in step with any change.

## How a change reaches the repos

- The workflows are called by tag (`@vX.Y.Z`). A workflow change does nothing until it's tagged; see "Release a
  workflow change" in `README.md`. A tag is immutable, so never move one.
- The community files and the Renovate preset are read from `main`, so they apply as soon as they merge.
- `deploy.yml` holds the Google Cloud login for every deploy, prod included. Change it with care.

## Tasks

mise provides the tools. Run `mise trust`, then `mise run install`.

- `mise run ci`: `check`, then `test`
- `mise run check`: Prettier check, and `actionlint` on the workflows
- `mise run format`: fix formatting

There are no tests. PR titles are conventional commits. Never push to `main`.
