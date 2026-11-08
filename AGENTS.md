# AGENTS.md

Guidance for coding agents working in this repository.

Follow `cyc:conventions` (the rules shared by every cyc-seattle repo).

## Overview

The files every cyc-seattle repo shares: the default community health files, the org profile (`profile/README.md`),
the reusable workflows (`ci.yml`, `release.yml`, `deploy.yml`, `pr-title.yml`), and the Renovate preset
(`renovate-config.json`). `README.md` says how a repo uses each workflow.

Repos pin the reusable workflows to a `vX.Y.Z` tag, and the Workload Identity provider accepts `deploy.yml` only at a
tag. A change here reaches a repo only once it's tagged and the repo moves its pin. A breaking change to a workflow's
inputs or behavior is a `feat!:`.

## Tasks

mise provides the tools. Run `mise trust`, then `mise run install`.

- `mise run ci`: `check`, then `test`
- `mise run check`: Prettier check, and `actionlint` on the workflows
- `mise run format`: fix formatting

There's no code to build and no tests.

## Working in this repo

PR titles are conventional commits. Never push to `main`.
