# devops-workflows

Reusable GitHub Actions workflows shared across small Python/Flask projects
(expense-analyzer, income-analyzer, ...). The pipeline logic lives here once;
each project repo keeps only its own source, `Dockerfile`, and `fly.toml`
(those are inherently per-project) plus a thin workflow file that calls into
this repo.

## What's here

- [.github/workflows/python-ci.yml](.github/workflows/python-ci.yml) —
  lint (`ruff`) → test (`pytest`) → `docker build` as a build-health check.
- [.github/workflows/fly-deploy.yml](.github/workflows/fly-deploy.yml) —
  `flyctl deploy` against a project's `fly.toml`.

Local dev tooling (`Makefile`, `.pre-commit-config.yaml`, `pyproject.toml`
ruff config) is **not** included here — that runs on a developer's machine,
not in CI, so it stays in each project repo (it's small and stable enough
that light duplication is cheap; see individual project repos).

## Versioning

Consumers should pin to a tag (`@v1`), never `@main`. An in-progress edit to
a workflow here would otherwise silently break every project mid-flight.
Bump a project's pin deliberately when you want it to pick up a pipeline
change:

```
git tag v1 && git push origin v1
```

When making a breaking change to an input/output, cut a new major tag
(`v2`) rather than moving `v1`.

## Usage in a consumer repo

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: <owner>/devops-workflows/.github/workflows/python-ci.yml@v1
    with:
      python-version: "3.11"
      # working-directory: "."          # defaults shown; override only if needed
      # requirements-file: "requirements-dev.txt"
      # dockerfile: "Dockerfile"
```

`.github/workflows/deploy.yml` (e.g. triggered manually or on tag push):
```yaml
name: Deploy
on:
  workflow_dispatch:

jobs:
  deploy:
    uses: <owner>/devops-workflows/.github/workflows/fly-deploy.yml@v1
    secrets:
      FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

`FLY_API_TOKEN` (`fly tokens create deploy` or `fly auth token`) must be set
as a repository secret on each consumer repo — it's not shared across repos
automatically.

## Requirements on the consumer repo

- `requirements-dev.txt` at `working-directory` installs both runtime and
  dev/lint/test deps (see any existing project for the pattern — it's a
  `-r requirements.txt` plus `ruff`/`pytest`).
- `pyproject.toml` with `[tool.ruff]` config (or defaults are fine).
- `tests/` with pytest-discoverable smoke tests that don't require live
  API keys — CI has none configured.
- A `Dockerfile` that builds standalone (no build secrets required).
- A `fly.toml` for the deploy workflow.

## Access from private repos

If a consumer repo is private and lives in a different GitHub account/org
than this repo, the caller needs permission to use workflows here — see
[Sharing actions and workflows with your organization](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#access-to-reusable-workflows).
If both repos are in the same org this generally works with default
permissions; if `devops-workflows` is public, any repo can reference it.
