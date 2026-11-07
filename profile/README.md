# CYC Community Sailing Center

CYC Community Sailing Center is a nonprofit in Seattle. Its volunteers write the code here, which runs the club's web app, sign-in, and the syncs between Clubspot, Google Workspace, and the club's database.

## What the club runs

- [`sail-team`](https://github.com/cyc-seattle/sail-team): the club's web app at [cycsail.team](https://cycsail.team), with plans, the roster, and links.
- [`platform`](https://github.com/cyc-seattle/platform): the shared Google Cloud setup, the club's database (Directus), and the jobs that sync it with Clubspot and Google Workspace.
- [`isthmia`](https://github.com/cyc-seattle/isthmia): sign-in for staff and families, and the older links portal that `sail-team` is replacing.
- [`gam`](https://github.com/cyc-seattle/gam): backup scripts for managing Google Workspace groups by hand when the sync breaks.
- [`awards`](https://github.com/cyc-seattle/awards): source files for the club's trophies and awards.

## Shared code and setup

- [`clubspot-sdk`](https://github.com/cyc-seattle/clubspot-sdk): a TypeScript library and command-line tool for Clubspot.
- [`gsuite-sdk`](https://github.com/cyc-seattle/gsuite-sdk): a TypeScript library for Google Workspace.
- [`commodore`](https://github.com/cyc-seattle/commodore): shared helpers for the command-line tools.
- [`ts-config`](https://github.com/cyc-seattle/ts-config): shared lint, format, and test settings.
- [`agent-skills`](https://github.com/cyc-seattle/agent-skills): instructions for the AI coding agents that help work on the repos.
- [`workplace`](https://github.com/cyc-seattle/workplace): checks out every repo side by side.
- [`.github`](https://github.com/cyc-seattle/.github): this profile, the contributing guide, and shared GitHub workflows.

## How to help

Read the [contributing guide](https://github.com/cyc-seattle/.github/blob/main/CONTRIBUTING.md). Then pick an open issue in a repo, or open a new one.
