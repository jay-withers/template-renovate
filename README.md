# template-renovate

Centralised [Renovate](https://docs.renovatebot.com/) configuration presets. Instead of maintaining a full `renovate.json` in every repository, each repo extends the shared presets defined here. Change the policy once, and every consuming repo picks it up.

Built on [`template-base-repo`](https://github.com/jay-withers/template-base-repo), so it also ships a dev container, pre-commit hooks, CI/CD and branch-protection scaffolding.

## Usage

Add a `renovate.json` (or `.github/renovate.json`) to a consuming repo:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>jay-withers/template-renovate"]
}
```

That single line pulls in [`default.json`](default.json), which wires up all the presets below.

## Presets

| Preset | Extend as | What it does |
| --- | --- | --- |
| **default** | `github>jay-withers/template-renovate` | The recommended everything-included config. Extends all presets below plus `config:recommended`, dependency dashboard, semantic commits and sign-off, and enables the `pre-commit` manager. |
| **automerge** | `github>jay-withers/template-renovate:automerge` | Auto-merges non-major dev dependencies, pins, digests and lockfile maintenance once CI passes. Majors always require review. |
| **schedule** | `github>jay-withers/template-renovate:schedule` | Batches updates for before 6am on Monday to reduce mid-week churn. |
| **docker** | `github>jay-withers/template-renovate:docker` | Pins image digests and groups Docker updates. |
| **github-actions** | `github>jay-withers/template-renovate:github-actions` | Pins Actions to commit SHAs and groups them. |
| **terraform** | `github>jay-withers/template-renovate:terraform` | Groups Terraform/Terragrunt providers and modules. |
| **npm** | `github>jay-withers/template-renovate:npm` | Groups npm dev vs production dependencies and `@types`. |

### Picking individual presets

You don't have to take everything. Compose only what you need:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "github>jay-withers/template-renovate:docker",
    "github>jay-withers/template-renovate:github-actions"
  ]
}
```

### Overriding

Extend the shared config, then override anything locally — later entries win:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>jay-withers/template-renovate"],
  "schedule": ["at any time"],
  "packageRules": [
    {
      "matchPackageNames": ["react", "react-dom"],
      "automerge": false
    }
  ]
}
```

## Getting started (developing this repo)

1. Open it in the dev container (VS Code: **Reopen in Container**, or GitHub Codespaces). The container runs `make install` on creation to wire up the pre-commit hooks.
2. Outside a dev container, install the hooks manually with `make install`.
3. Configure the GitHub-side settings that can't be templated as files (see below): `make protect-branch CHECKS="$(printf 'pre-commit / Pre-commit\nvalidate')"`.

## Commands

Run `make` (or `make help`) to list the available targets:

```bash
make install           # install pre-commit hooks (run once after cloning)
make validate          # validate all Renovate presets with renovate-config-validator
make lint              # run all pre-commit hooks against every file
make protect-branch    # configure GitHub repo settings (auto-merge, branch protection)
```

Validate presets directly without make:

```bash
npx --yes --package renovate -- renovate-config-validator --strict *.json
```

## Commit messages

Commits must follow [Conventional Commits](https://www.conventionalcommits.org/), enforced by commitlint at commit-msg time. The commit type drives the automatic version bump on merge. Examples:

```text
feat: add terraform preset
fix: correct docker digest grouping
chore: bump pre-commit hooks
```

## CI/CD

Workflows are prefixed `ci-` (pull-request checks) or `cd-` (post-merge delivery):

- **`.github/workflows/ci-lint.yml`** — runs all pre-commit hooks on PRs to `main` via the shared reusable workflow `jay-withers/template-pipelines/.github/workflows/pre-commit.yml`. Status check: `pre-commit / Pre-commit`.
- **`.github/workflows/ci-validate.yml`** — validates every root `*.json` preset with `renovate-config-validator --strict`. Status check: `validate`.
- **`.github/workflows/cd-tag.yml`** — on every merge to `main`, creates a semver tag and matching GitHub release from the Conventional Commits since the last release (default bump: patch), via `jay-withers/template-pipelines/.github/workflows/release.yml`.

## Configuring GitHub

Some settings can't be templated as files and need to be set once per repo via the GitHub API. With the [`gh` CLI](https://cli.github.com) authenticated as an admin on the repo:

```bash
make protect-branch CHECKS="$(printf 'pre-commit / Pre-commit\nvalidate')"
```

This runs `scripts/protect-branch.sh` and is idempotent. It enables repository **auto-merge** (which the automerge preset's `platformAutomerge` depends on) and **delete branch on merge**, then replaces all rulesets with one on the target branch requiring the given status checks and approving reviews, with the Renovate GitHub App and the repo **Admin** role exempted as bypass actors.

GitHub only honours those bypass actors on **organisation-owned** repos, so on a personal (user-owned) repo the script defaults required reviews to `0` instead of `1` — otherwise Renovate's own auto-merge would be blocked forever waiting for a review it can't give itself. Override with `APPROVALS_REQUIRED=<n>` if needed; see `scripts/protect-branch.sh` for the full reasoning.

`CHECKS` is a **newline-separated** list of status-check contexts (newline, not space, because a context name can itself contain spaces like `pre-commit / Pre-commit`). This repo has two checks — `pre-commit / Pre-commit` and `validate` — so pass both. Confirm exact names with `gh pr checks`.

## Structure

```text
.devcontainer/
  devcontainer.json    # dev container (ghcr.io/jay-withers/dev-containers/base)
.github/
  workflows/
    ci-lint.yml        # lints all files on PRs to main (reusable workflow)
    ci-validate.yml    # validates Renovate presets on PRs to main
    cd-tag.yml         # auto-tags + releases on merge to main
default.json           # the recommended shared preset (extend this)
automerge.json         # auto-merge policy preset
schedule.json          # update schedule preset
docker.json            # Docker / container preset
github-actions.json    # GitHub Actions preset
terraform.json         # Terraform / Terragrunt preset
npm.json               # Node / npm preset
renovate.json          # this repo dogfoods its own config
.editorconfig          # baseline editor settings
.gitattributes         # git-level LF normalization
.pre-commit-config.yaml
commitlint.config.js   # commitlint (Conventional Commits) config
scripts/
  protect-branch.sh    # one-time GitHub settings (auto-merge, branch protection ruleset)
CLAUDE.md              # guidance for Claude Code
LICENSE
Makefile
```
