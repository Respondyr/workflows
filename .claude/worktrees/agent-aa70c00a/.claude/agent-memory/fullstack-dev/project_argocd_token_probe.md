---
name: ArgoCD Token Health Probe
description: CronJob + alert added to detect expired ArgoCD CI tokens before they block deploys
type: project
---

ArgoCD CI token health probe implemented in PR #47. CronJob runs every 30min in argocd namespace, probes API with Bearer token, alerts via Loki/Alertmanager on failure.

**Why:** On 2026-04-07, CI tokens expired silently on both clusters blocking all deployments with zero notification. This was the root cause of the deploy outage.

**How to apply:** If ArgoCD deploy issues arise, check whether the `ArgocdTokenAuthFailure` alert has fired. Token rotation requires `argocd account generate-token --account ci --expires-in 0` followed by updating the value in AWS Secrets Manager at `{env}/undercurrentsmb/master-list`.
