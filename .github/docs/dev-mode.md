# Dev Mode — CI Minutes Optimization

## Overview

All reusable workflows (`security.yml`, `go-service.yml`, `python-service.yml`, `frontend.yml`) support a `dev_mode` input that skips expensive CI steps during active development.

With 2,000 GitHub Actions minutes/month and 10+ repos, full pipelines burn through the budget fast. Dev mode cuts per-run time from ~10 min to ~2-4 min.

## What Gets Skipped

| Step | Normal | Dev Mode |
|------|--------|----------|
| Gitleaks (secrets scan) | Runs | **Skipped** |
| Dep scan (govulncheck / pip-audit / npm audit) | Runs | **Skipped** |
| Trivy image scan | Runs | **Skipped** |
| Docker build on PRs | Builds + pushes | **Skipped entirely** |
| Docker platform targets | linux/amd64 + linux/arm64 | **linux/amd64 only** |
| QEMU setup (for arm64) | Runs | **Skipped** |
| Go vet / test / build | Runs | Runs (always) |
| Lint (ruff / eslint) | Runs | Runs (always) |
| pytest / go test | Runs | Runs (always) |

## Usage

In your repo's `.github/workflows/ci.yml`, pass `dev_mode: true`:

```yaml
jobs:
  ci:
    uses: Travbz/undercurrent-workflows/.github/workflows/go-service.yml@v0
    with:
      dev_mode: true          # <-- add this line
      service_name: my-service
      has_docker: true
      # ... other inputs
    secrets: inherit
```

## Estimated Savings

| Scenario | Before | Dev Mode | Saved |
|----------|--------|----------|-------|
| Go service PR | ~10 min | ~2 min | ~8 min |
| Go service main push | ~10 min | ~4 min | ~6 min |
| Python service PR | ~6 min | ~2 min | ~4 min |
| Frontend PR | ~5 min | ~3 min | ~2 min |

## Turning Off Dev Mode

When ready for production, set `dev_mode: false` or remove the line (defaults to `false`). Full security scanning, multi-arch builds, and image scanning resume automatically.

## Affected Workflows

- **security.yml** — all 3 jobs gated: `gitleaks`, `dep-scan`, `trivy`
- **go-service.yml** — security skipped, Docker skipped on PRs, single-arch on main, no image-scan
- **python-service.yml** — same as go-service
- **frontend.yml** — security skipped
