# Contributing

This guide applies to every cyc-seattle repo. A repo's own `README.md` and `AGENTS.md` add what is specific to it.

## Set up

1. Install [mise](https://mise.jdx.dev/getting-started.html).
2. Install Docker. Docker Desktop, OrbStack, and colima all work.
3. In the repo, run `mise run install`.

## Tasks

Every repo has the same five mise tasks:

| Task               | Does                                         |
| ------------------ | -------------------------------------------- |
| `mise run install` | Installs the tools and dependencies          |
| `mise run format`  | Formats the files and fixes what it can      |
| `mise run check`   | Checks formatting, lint, and types           |
| `mise run test`    | Runs the tests                               |
| `mise run ci`      | Runs `check` and `test`, the same as CI does |

CI runs `mise run ci`. Run it before you open a PR.

## Pull requests

Who can open a PR depends on the repo:

- **App repos** (`platform`, `isthmia`): only named contributors with write access. Open the PR from a branch in the repo, not from a fork. Their CI uses secrets, and GitHub gives a fork no secrets.
- **Library and org repos**: anyone. Open the PR from a fork.

PRs merge by squash.

## Tests

1. CI must pass before a PR merges.
2. A change in behavior comes with a test that fails without it. A trivial change, such as a typo, a rename, or a docs edit, needs none.
3. The deprecated repos run the tests they have and get no new ones.

There are two kinds of test:

- **Unit tests** check one module and use no network.
- **Integration tests** are written in Gherkin. Each scenario checks one behavior.

## Plans

Plans live on issues. If a change needs a decision, post a plan on its issue before you write the code. Most changes need no plan.

## Commits

Write each PR title as a [conventional commit](https://www.conventionalcommits.org/). The squash merge uses the PR title as the commit subject, and semantic-release reads it to pick the release. A required check rejects any other title.

| Type                                           | Use it for                    | Release |
| ---------------------------------------------- | ----------------------------- | ------- |
| `fix:`                                         | A bug fix                     | Patch   |
| `feat:`                                        | A new feature                 | Minor   |
| `feat!:`, `fix!:`                              | A change that breaks callers  | Major   |
| `chore:`, `docs:`, `ci:`, `test:`, `refactor:` | Work that changes no behavior | None    |

Write the description in the imperative.

- Good: `feat: add the roster export`
- Bad: `Added roster export`

## Agents

If you work with a coding agent:

- The shared skills are in [`cyc-seattle/agent-skills`](https://github.com/cyc-seattle/agent-skills).
- A repo's `AGENTS.md` holds the facts about that repo.
