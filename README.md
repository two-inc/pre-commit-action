# pre-commit-action

Composite action that runs [pre-commit](https://pre-commit.com/) hooks against the PR diff and posts a sticky comment with the result.

Replaces the `style.yaml` boilerplate currently duplicated across ~20 two-inc repos.

## Usage

Copy this workflow verbatim into `.github/workflows/style.yaml`:

```yaml
name: Pre-commit

on:
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: pre-commit-${{ github.ref_name }}
  cancel-in-progress: true

jobs:
  pre-commit:
    runs-on: ${{ vars.RUNNER_STANDARD }}
    timeout-minutes: 20
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: two-inc/pre-commit-action@main
        with:
          python-version: "3.12"
```

## Inputs

| Input            | Default               | Description                           |
| ---------------- | --------------------- | ------------------------------------- |
| `python-version` | `3.12`                | Python version to install             |
| `github-token`   | `${{ github.token }}` | Token used to post the sticky comment |

## Requirements

The consumer repo must declare `pre-commit` in its pyproject `dev` dependency group, e.g.:

```toml
[dependency-groups]
dev = [
  "pre-commit>=4.5.1",
  # ...
]
```

The action runs `uv run --only-group dev pre-commit run ...`, which installs only the `dev` group (not the project itself) and uses the pre-commit version declared there. This means the pre-commit version is tracked alongside the project's other tooling, rather than being pinned in the workflow.

## Behaviour

- Checks out the PR with full history (`fetch-depth: 0`) so pre-commit can diff against `origin/${{ github.base_ref }}`.
- Installs `uv` with caching enabled, sets up Python.
- Runs `uv run --only-group dev pre-commit run --from-ref ... --to-ref ...` against the PR diff. The project itself is not installed — only the `dev` dependency group, which should contain pre-commit.
- Posts the result as a sticky PR comment identified by the `pre-commit` header marker — the comment is updated in place across pushes instead of deleted and recreated.
- Exits with pre-commit's exit code so the job reflects pass/fail.

## Migration notes

The action deliberately drops two things that appeared in many existing `style.yaml` files:

- **GCP Artifact Registry auth** — not needed. The `two` / `two-pypi` Artifact Registry indices used by two-inc Python repos are anonymously readable, so `uv` can resolve dev-group deps like `two-dev-cli` without authenticating.
- **Full project `uv sync`** — not needed. `--only-group dev` installs just the dev group; pre-commit itself manages each hook's isolated environment.
