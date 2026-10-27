# Aohzan/.github

Shared GitHub configuration for the [Aohzan](https://github.com/Aohzan) Home Assistant custom integrations.

- Default community health files: issue forms, pull request template, funding.
- Reusable workflows, called with `@v1`:

| Workflow | Purpose | Caller requirements |
|---|---|---|
| `ha-ci.yaml` | Ruff, translations, pytest, Home Assistant constraints | `pyproject.toml` + `uv.lock`, `.python-version`, `scripts/check_translations.py`, `scripts/sync_requirements.py` |
| `ha-validate.yaml` | hassfest and HACS | — |
| `ha-release.yaml` | semantic-release | `.releaserc`, `scripts/release.sh`; `contents`, `issues`, `pull-requests` write |
| `dependabot-auto-merge.yaml` | Auto-merge of non-major Dependabot updates | `contents`, `pull-requests` write; branch ruleset with required checks |

## Caller example

`.github/workflows/ci.yaml`:

```yaml
name: CI

on:
  push:
  pull_request:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  ci:
    uses: Aohzan/.github/.github/workflows/ha-ci.yaml@v1
  validate:
    uses: Aohzan/.github/.github/workflows/ha-validate.yaml@v1
  release:
    needs: [ci, validate]
    if: github.event_name == 'push' && github.ref_name == github.event.repository.default_branch
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: Aohzan/.github/.github/workflows/ha-release.yaml@v1
    secrets: inherit
```

## Repository settings

- `RELEASE_TOKEN`: fine-grained personal access token (contents and pull requests read/write), stored as an Actions **and** a Dependabot secret. Without it the workflows fall back to `GITHUB_TOKEN`, but a release commit can't be pushed through the branch ruleset and an auto-merged Dependabot update doesn't trigger a release.
- "Allow auto-merge" enabled.
- Ruleset on the default branch: required status checks `ci / Lint`, `ci / Tests`, `ci / Home Assistant constraints`, `validate / Hassfest`, `validate / HACS`, with the repository admin role as bypass actor.
