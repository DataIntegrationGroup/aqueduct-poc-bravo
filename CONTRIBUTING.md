# Contributing to Aqueduct

`main` is the trunk branch. All feature work branches off `main` and merges back
via pull request.

## Workflow overview

1. Create a branch from Jira using the supplied branch name.
2. Make changes; keep your branch up to date with `main` via rebase if needed.
3. Open a PR against `main` with a [Conventional Commit](https://www.conventionalcommits.org/) title.
4. Wait for CI and one approval from another contributor.
5. Squash-merge your own PR after approval.

---

## Branching

Create branches from Jira tickets. Use the branch name Jira provides — it links the
PR to the ticket automatically.

| | Example |
|---|---|
| Jira ticket | `ST2DAT-100` |
| Branch | `ST2DAT-100-add-release-please` |

Branch off latest `main`:

```bash
git checkout main
git pull
git checkout -b ST2DAT-100-add-release-please
```

---

## Linear history

`main` requires a linear history — no merge commits. Use **squash merge** for PRs.

While your PR is open, sync with `main` by rebasing (not merging `main` into your branch):

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```

---

## Commits and PR titles

### PR title (required)

The PR title must be a Conventional Commit describing the **overall theme** of the
change. A [pr-title-lint](.github/workflows/pr-title-lint.yml) workflow enforces this.

Because we squash-merge with the PR title as the commit message, the title becomes
what lands on `main` and what release-please uses for the changelog.

| | Example |
|---|---|
| Branch | `ST2DAT-100-add-release-please` |
| PR title | `chore(ci): add release-please` |

**Allowed types:** `feat`, `fix`, `perf`, `docs`, `chore`, `refactor`, `test`, `ci`,
`build`, `style`, `revert`, `deps`

**Rules:**
- Subject starts with a lowercase letter
- Scope is optional: `feat(hydrovu): add location filtering`
- Breaking changes: `feat!:` or `fix!:` in the title

### Branch commits (optional)

Commits on your branch can be informal (`fix typo`, `address review`). Only the PR
title matters for history and releases. Keeping branch commits reasonably clean still
helps reviewers.

---

## Pull requests

1. Push your branch and open a PR against `main`.
2. Fill in the description — what changed and why.
3. Wait for required checks:
   - `lint` and `test` ([CI](.github/workflows/ci.yml))
   - `Validate PR title` ([pr-title-lint](.github/workflows/pr-title-lint.yml))
4. Request review from another contributor. **One approval is required.**
5. After approval and green checks, **merge your own PR** using **Squash and merge**.

Do not merge your own PR without approval.

---

## Releases

[release-please](https://github.com/googleapis/release-please) maintains a Release PR
on `main`. Only admins merge release PRs.

---

## Local development

See [README.md](README.md) for setup, running tests, and Dagster.

Before pushing:

```bash
uv sync --group dev
uv run pre-commit run --all-files
uv run pytest
```
