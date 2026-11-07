# CYC Seattle

Volunteers write the code here for two Seattle sailing nonprofits:

- [CYC of Seattle](https://www.cycseattle.org/)
- [CYC Community Sailing Center](https://www.cyccommunitysailing.org/)

It runs the web app, sign-in, and the syncs between Clubspot, Google Workspace, and the database.

## What runs today

- [`sail-team`](https://github.com/cyc-seattle/sail-team): the web app at [cycsail.team](https://cycsail.team), with plans, the roster, and links.
- [`platform`](https://github.com/cyc-seattle/platform): the shared Google Cloud setup, the database (Directus), sign-in (Authentik), and the jobs that sync it with Clubspot and Google Workspace.
- [`isthmia`](https://github.com/cyc-seattle/isthmia): the older links portal, which `sail-team` is replacing. Sign-in has moved to `platform`.
- [`gam`](https://github.com/cyc-seattle/gam): backup scripts for managing Google Workspace groups by hand when the sync breaks.
- [`awards`](https://github.com/cyc-seattle/awards): source files for CYC's and the Sailing Center's trophies and awards.

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
