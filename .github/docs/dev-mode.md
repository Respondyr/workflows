# Dev Mode — CI Minutes Optimization

## Overview

`dev_mode` is an opt-in input that skips expensive scanning steps to save
GitHub Actions minutes during active development. It exists on two reusable
workflows in this repo:

- **security.yml** — `dev_mode: true` skips both jobs: `dep-scan`
  (govulncheck / pip-audit / npm audit) and `trivy` (image scan).
- **e2e-tests.yml** — declares the input, but no step currently consumes it.

The monolithic `go-service.yml` / `python-service.yml` / `frontend.yml`
workflows this doc originally covered were replaced by the split
`go-ci.yml` / `docker-build.yml` / `deploy.yml` pipeline (see
[CICD.md](../../CICD.md)). Those replacements do not take `dev_mode`, and
gitleaks now runs inside `go-ci.yml`, not `security.yml`.

## Usage

Pass `dev_mode: true` where a workflow calls `security.yml`:

```yaml
jobs:
  security:
    uses: Respondyr/workflows/.github/workflows/security.yml@security/v0
    with:
      language: go
      dev_mode: true          # skip dep-scan + trivy
    secrets: inherit
```

## Turning Off Dev Mode

Set `dev_mode: false` or remove the line (defaults to `false`). Full
dependency and image scanning resume automatically.
