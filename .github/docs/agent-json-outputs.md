# Future Enhancement: Agent-Readable JSON Output Contract

**Status: proposed, not built.** Design captured 2026-08-01 so the context survives.

## Why

Agents (Claude Code in CI, the Slack DevOps agent, anything summoned into a
pipeline) are only as useful as what they can read back from a run. Today our
reusable workflows communicate through exit codes and logs. An agent that wants
to know "what image did this build, where did it deploy, why did it fail" has
to grep log text. The fix is a structured output contract: every reusable
workflow emits machine-readable JSON alongside its human-facing output.

## Design

### 1. Shared emitter composite action (`actions/emit`)

Define the emit step once, not in every workflow. New top-level `actions/` dir
(composite actions can live anywhere in the repo; only reusable workflows are
locked to `.github/workflows/`):

```
workflows/
├── .github/workflows/     # reusable workflows (flat, GitHub requirement)
├── .github/docs/          # this doc
├── actions/
│   └── emit/action.yml    # shared JSON emitter
└── schemas/               # JSON Schema per output contract
    └── build-manifest.schema.json
```

```yaml
# actions/emit/action.yml
name: emit-json
description: Validate, compact, and emit a JSON output + human summary
inputs:
  json:
    description: JSON string, or path to a JSON file
    required: true
  schema:
    description: Optional schema path to validate against
    required: false
    default: ""
outputs:
  value:
    value: ${{ steps.emit.outputs.value }}
runs:
  using: composite
  steps:
    - id: emit
      shell: bash
      env:
        RAW: ${{ inputs.json }}
        SCHEMA: ${{ inputs.schema }}
      run: |
        if [ -f "$RAW" ]; then JSON=$(cat "$RAW"); else JSON="$RAW"; fi
        echo "$JSON" | jq empty
        if [ -n "$SCHEMA" ]; then
          check-jsonschema --schemafile "$SCHEMA" <(echo "$JSON")
        fi
        echo "value=$(echo "$JSON" | jq -c .)" >> "$GITHUB_OUTPUT"
        { echo '```json'; echo "$JSON" | jq .; echo '```'; } >> "$GITHUB_STEP_SUMMARY"
```

The `env:` indirection is load-bearing: interpolating `${{ inputs.json }}`
directly into the script body is a shell injection hole once agent-generated
content flows through it.

### 2. Thread outputs up three layers

Step output -> job `outputs:` -> `on.workflow_call.outputs`. Callers and agents
read `needs.<job>.outputs.<name>` and parse with `fromJSON()`. Outputs are
single-line strings, so always compact with `jq -c`.

First target: `deploy.yml` emits a `manifest` output after the BUILD step:

```json
{
  "service": "reviews",
  "image": "987213268382.dkr.ecr.us-east-1.amazonaws.com/reviews",
  "sha_tag": "abc1234",
  "digest": "sha256:...",
  "dev_synced": true,
  "release_version": "v1.4.2"
}
```

### 3. Artifacts for large payloads

Outputs cap at 1MB per step, 50MB per run. Test reports, SBOMs, and scan
results upload as deterministically named artifacts (`build-manifest`,
`test-report`); the small JSON output carries the artifact name so the
contract points to the payload. Agents fetch via `gh run download`.

### 4. Failure signals

`echo "::error file=...,line=...::message"` produces Checks API annotations
agents read instead of grepping logs. `$GITHUB_STEP_SUMMARY` stays the human
twin of every machine output.

## Gotchas (learned before building)

- **Ref resolution:** inside a reusable workflow, `uses: ./actions/emit`
  resolves against the CALLER's checkout, not this repo. Reference the
  composite action by full path with a ref
  (`Respondyr/workflows/actions/emit@v1`). Internal refs then need bumping at
  release time; `semver-release.yml` + `reconcile-floats.yml` already manage
  floating major tags here, so extend that machinery rather than adding new.
- **Silent drops:** any output whose value contains a masked secret is dropped
  without error; the job output arrives empty. Keep secrets out of manifests.
- **`fromJSON()` fails the whole expression** on invalid JSON. Validate at
  emit time (`jq empty`) so bad JSON fails the producing job, not the consumer.

## Adoption sketch

1. `actions/emit` + `schemas/` + manifest output from `deploy.yml` build step.
2. Wire `go-ci.yml` / `python-ci.yml` / `frontend-ci.yml` to emit structured
   test results the same way.
3. `AGENTS.md` at repo root documenting the call/read contract for agents.
