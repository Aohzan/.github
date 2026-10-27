# Aohzan/.github

Shared GitHub configuration for the [Aohzan](https://github.com/Aohzan) Home Assistant custom integrations.

- Default community health files: issue forms, pull request template, funding.
- Reusable workflows, called with `@v2` (`@v1` is frozen):

| Workflow | Purpose | Caller requirements |
|---|---|---|
| `ha-ci.yaml` | Ruff, translations, pytest, Home Assistant constraints | `pyproject.toml` + `uv.lock`, `.python-version`, `scripts/check_translations.py`, `scripts/sync_requirements.py` |
| `ha-validate.yaml` | hassfest and HACS | — |
| `ha-release.yaml` | semantic-release | `.releaserc`, `scripts/release.sh`; `contents`, `issues`, `pull-requests` write |
| `dependabot-merge.yaml` | Merge of non-major Dependabot updates, from a CI job that needs the checks | `contents`, `pull-requests` write |
| `dependabot-auto-merge.yaml` | v1 only: GitHub auto-merge, which a token bypassing the ruleset merges without waiting for the checks | — |

## Caller example

`.github/workflows/ci.yaml`:

```yaml
name: CI

on:
  push:
    branches: [main]
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
    uses: Aohzan/.github/.github/workflows/ha-ci.yaml@v2
  validate:
    uses: Aohzan/.github/.github/workflows/ha-validate.yaml@v2
  release:
    needs: [ci, validate]
    if: github.event_name == 'push' && github.ref_name == github.event.repository.default_branch
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: Aohzan/.github/.github/workflows/ha-release.yaml@v2
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}

  dependabot:
    needs: [ci, validate]
    if: >-
      github.event_name == 'pull_request'
      && github.actor == 'dependabot[bot]'
      && github.event.pull_request.head.repo.full_name == github.repository
    permissions:
      contents: write
      pull-requests: write
    uses: Aohzan/.github/.github/workflows/dependabot-merge.yaml@v2
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}
```

In `.github/dependabot.yml`, use `versioning-strategy: lockfile-only` for uv: the lower bounds of `pyproject.toml` (the oldest supported Home Assistant) must not move with the lock.

## Repository settings

- `RELEASE_TOKEN`: fine-grained personal access token (contents, pull requests and workflows read/write: Dependabot updates of the GitHub Actions change workflow files), stored as an Actions **and** a Dependabot secret. Without it the workflows fall back to `GITHUB_TOKEN`, but a release commit can't be pushed through the branch ruleset and an auto-merged Dependabot update doesn't trigger a release.
- "Allow auto-merge" enabled.
- Ruleset on the default branch: required status checks `ci / Lint`, `ci / Tests`, `ci / Home Assistant constraints`, `validate / Hassfest`, `validate / HACS`, with the repository admin role as bypass actor.
