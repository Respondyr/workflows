# UndercurrentSMB CI/CD Architecture

## Principles

1. **Build once, promote the same artifact.** Docker images are built once on main merge, tested in dev, then retagged for prod. No double builds.
2. **Convention over configuration.** The language tells you how to test. No test paths to configure.
3. **No bot commits to main.** Deployments use ArgoCD API calls, not git commits to values files. No `[skip ci]` hacks, no push race conditions.
4. **PR = fast feedback. Main = thorough pipeline.** PRs run lint + test only. Main merge runs security, Docker build, dev deploy, verification, release, and prod deploy.

## Flow

```
PR opened
    |
    v
 go-ci / python-ci / frontend-ci
 (lint, test, build — no Docker, no deploy)
    |
    PR flow ends.


Merge to main
    |
    v
 deploy.yml (unified pipeline)
    |
    1. TEST      — language-specific (go test / pytest / npm test)
    2. SECURITY  — gitleaks + dependency scan
    3. BUILD     — Docker image → ECR (:sha + :latest)
    4. DEV       — ArgoCD: set tag → sync → verify healthy
    5. VERIFY    — e2e tests against dev (if has_e2e)
    6. RELEASE   — conventional commits → semver → GitHub Release
    7. RETAG     — ECR manifest copy :sha → :vX.Y.Z
    8. PROD      — ArgoCD: set tag → sync → verify healthy
```

## Workflow Files

| File | Purpose | When |
|------|---------|------|
| `go-ci.yml` | Go PR checks: vet, test, build | PR only |
| `python-ci.yml` | Python PR checks: lint, test | PR only |
| `frontend-ci.yml` | Frontend PR checks: lint, test, build | PR only |
| `deploy.yml` | Unified deploy pipeline (all languages) | Main merge only |
| `security.yml` | Gitleaks + dependency scanning | Called by deploy.yml |
| `e2e-tests.yml` | Playwright e2e test runner | Called by deploy.yml verify step |
| `release.yml` | Self-release for this repo (per-component tags) | Main push (this repo only) |

## Test Convention

No test paths needed. The language determines the command:

| Language | Test command | Discovery |
|----------|-------------|-----------|
| Go | `go test ./...` | All `*_test.go` files |
| Python | `pytest tests/ -v` | All `test_*.py` in `tests/` |
| Node | `npm test` | Defined in `package.json` |

If a repo has no tests, the step either passes silently (Go) or is skipped via `has_tests: false`.

## Caller Pattern

Every service repo has a single `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: write
  id-token: write
  security-events: write

jobs:
  ci:
    if: github.event_name == 'pull_request'
    uses: UndercurrentSMB/undercurrent-workflows/.github/workflows/go-ci.yml@main
    with:
      has_auth_dep: true
      build_target: "./cmd/server"
    secrets: inherit

  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: UndercurrentSMB/undercurrent-workflows/.github/workflows/deploy.yml@main
    with:
      language: go
      service_name: my-service
      has_docker: true
      has_auth_dep: true
      build_target: "./cmd/server"
    secrets: inherit
```

## deploy.yml Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `language` | string | required | `go`, `python`, or `node` |
| `service_name` | string | required | ECR repo name + ArgoCD app name |
| `has_docker` | bool | `true` | Build/push Docker image |
| `argocd_enabled` | bool | `true` | Deploy via ArgoCD API |
| `has_e2e` | bool | `false` | Run e2e tests after dev deploy |
| `go_version` | string | `1.25` | Go version |
| `has_auth_dep` | bool | `false` | Clone auth-go for replace directive |
| `build_target` | string | `./cmd/server` | Go build target |
| `python_version` | string | `3.12` | Python version |
| `install_method` | string | `pip` | `pip` or `editable` |
| `src_path` | string | `src/` | Python lint path |
| `has_tests` | bool | `true` | Run language-specific tests |
| `node_version` | string | `20` | Node.js version |
| `build_command` | string | `npm run build` | Node build command |

## Repo Categories

**Full pipeline (Docker + ArgoCD):** accounts, billing, brand, comms, reviews, ai

**Docker only (no ArgoCD):** deploy-listener (`argocd_enabled: false`)

**Library (no Docker, no deploy):** auth-go, intel, outreach (`has_docker: false`)

## Prerequisites

- **AWS OIDC role** (`AWS_ROLE_ARN` secret) with ECR push/pull and Secrets Manager read
- **ArgoCD tokens** in AWS Secrets Manager:
  - `dev/undercurrentsmb/master-list` → `ARGOCD-DEV-TOKEN`
  - `prod/undercurrentsmb/master-list` → `ARGOCD-PROD-TOKEN`
- **GitHub App** (`GH_APP_ID` + `GH_APP_PRIVATE_KEY`) for private dep access
- **ECR repositories** matching `service_name` for each Docker service
