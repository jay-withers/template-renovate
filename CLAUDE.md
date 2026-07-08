# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo does

A centralised [Renovate](https://docs.renovatebot.com/) configuration repo. Instead of every repository carrying its own full `renovate.json`, each repo extends the shared presets defined here (`extends: ["github>jay-withers/template-renovate"]`). Policy — schedule, auto-merge rules, ecosystem grouping — is changed once here and inherited everywhere. The repo also carries the language-agnostic scaffolding it was derived from `jay-withers/template-base-repo` (dev container, pre-commit hooks, CI/CD, branch protection).

## Presets

Each preset is a self-contained JSON file at the repo root, consumed via `github>jay-withers/template-renovate:<name>` (the bare repo reference loads `default.json`):

- **default.json** — the recommended everything-included config. Extends `config:recommended`, the dependency dashboard, semantic commits, git sign-off, and all the presets below. Also enables the `pre-commit` manager so derived repos get frozen-hook updates for free.
- **automerge.json** — auto-merges non-major dev deps, pins, digests and lockfile maintenance once CI is green (`platformAutomerge`); majors always require a human.
- **schedule.json** — batches updates for `before 6am on monday`.
- **docker.json** — pins image digests, groups Docker updates.
- **github-actions.json** — pins Actions to commit SHAs (`helpers:pinGitHubActionDigests`), groups them.
- **terraform.json** — groups Terraform/Terragrunt providers and modules.
- **npm.json** — groups npm dev vs production dependencies and `@types`.

Consumers override the shared config by setting options locally after the `extends` — later config wins (scalars replace, `packageRules` concatenate with later rules taking precedence).

`renovate.json` in this repo dogfoods the shared config (`extends: ["local>jay-withers/template-renovate"]`) plus `autoApprove`.

### Editing presets

- Keep each preset a standalone, valid Renovate config object with a `$schema` and a `description`.
- Validate before pushing: `make validate` (or `npx --yes --package renovate -- renovate-config-validator --strict <files>`).
- Every root-level `*.json` is validated in CI (see below). When you add a new preset file, add a matching row to the README table and the presets list above.
- `renovate-config-validator` does **not** resolve remote `extends` over the network, so `default.json`'s self-references (`github>jay-withers/template-renovate:...`) validate offline.

## Dev container

Built around `.devcontainer/devcontainer.json` (image `ghcr.io/jay-withers/dev-containers/base:latest`), which runs `make install` on creation to wire up the pre-commit hooks. Prefer working inside the container so tooling versions match CI.

## Commands

`make` with no target prints the self-documenting help (the default goal).

```bash
make install           # install pre-commit hooks (run once after cloning)
make validate          # validate all Renovate presets with renovate-config-validator
make lint              # run all pre-commit hooks against every file
make protect-branch    # configure GitHub repo settings (auto-merge, branch protection) — override BRANCH/CHECKS to match this repo's checks
```

## Commit messages

Commits must follow [Conventional Commits](https://www.conventionalcommits.org/) — enforced by commitlint at commit-msg time. Examples: `feat: add terraform preset`, `fix: correct docker digest grouping`, `chore: bump pre-commit hooks`.

## Pre-commit config

Hooks are in `.pre-commit-config.yaml`, pinned by commit SHA with the tag as a frozen comment: `pre-commit/pre-commit-hooks` basics, `gitleaks` (secrets), `actionlint` (workflows), `shellcheck` (shell), and `commitlint` (at the `commit-msg` stage). The Renovate `pre-commit` manager (enabled via `default.json`) keeps these revisions up to date.

## CI

Workflows are prefixed `ci-` (pull-request checks) or `cd-` (post-merge delivery):

- **ci-lint** (`.github/workflows/ci-lint.yml`): runs all linters on PRs to `main` via the reusable workflow `jay-withers/template-pipelines/.github/workflows/pre-commit.yml`. Its status-check context is `pre-commit / Pre-commit` (`<caller job id> / <reusable job name>`).
- **ci-validate** (`.github/workflows/ci-validate.yml`): runs `renovate-config-validator --strict` over every root `*.json` preset on PRs to `main`. Its status-check context is `validate`.
- **cd-tag** (`.github/workflows/cd-tag.yml`): auto-creates a semver tag and matching GitHub release on every merge to `main` from the Conventional Commits since the last release, via `jay-withers/template-pipelines/.github/workflows/release.yml` (default bump: patch).

## GitHub repo settings

`scripts/protect-branch.sh` (run via `make protect-branch`) sets the platform state that can't live in files: repo-level auto-merge (required for the automerge preset's `platformAutomerge`), delete-branch-on-merge, and a ruleset on the target branch requiring the given status checks plus approving reviews, with the Renovate GitHub App and the repo Admin role exempted as bypass actors. It clears existing rulesets first, so re-runs replace rather than accumulate.

GitHub only honours ruleset bypass actors on **organisation-owned** repos — on a personal (user-owned) repo the Renovate app exemption is silently ignored, so a required-review rule blocks Renovate's auto-merge forever. The script detects this and defaults `APPROVALS_REQUIRED` to `0` for user-owned repos (`1` for orgs); override with `APPROVALS_REQUIRED=<n>` if you add other human collaborators and want review enforced (Renovate itself will then need an auto-approve app, e.g. `renovate-approve`, to ever merge). This is safe even for public repos: merging still requires write access, which forks/outside contributors don't have, so dropping the approval count doesn't grant any new capability.

**This repo has two required checks**, so override the default `CHECKS` (which is just `pre-commit / Pre-commit`) to include `validate`:

```bash
make protect-branch CHECKS="$(printf 'pre-commit / Pre-commit\nvalidate')"
```

`CHECKS` is newline-separated because a context name can itself contain spaces (e.g. the reusable-workflow context above). Confirm exact context names with `gh pr checks`.
