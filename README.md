# template-renovate

Centralised [Renovate](https://docs.renovatebot.com/) configuration presets. Instead of maintaining a full `renovate.json` in every repository, each repo extends the shared presets defined here. Change the policy once, and every consuming repo picks it up.

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
| **default** | `github>jay-withers/template-renovate` | The recommended everything-included config. Extends all presets below plus `config:recommended`, dependency dashboard, semantic commits and sign-off. |
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

## Contributing

- Each preset is a self-contained JSON file at the repo root.
- Every `*.json` file is validated in CI by [`renovate-config-validator`](.github/workflows/validate.yml) on push and PR.
- Validate locally before pushing:

  ```sh
  npx --yes --package renovate -- renovate-config-validator --strict *.json
  ```
