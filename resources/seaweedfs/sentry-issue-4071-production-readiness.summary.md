---
source: https://github.com/getsentry/self-hosted/issues/4071
component: seaweedfs
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

GitHub Issue #4071 in the Sentry self-hosted repository characterizes SeaweedFS as an "Unstable local S3 provider" and raises the absence of a zero-downtime backup/restore strategy as a core production readiness gap. The issue was opened by a Sentry user evaluating SeaweedFS as Sentry's embedded object storage. The issue remains open, indicating no resolution was found.

## Key Points

- Primary concern: no zero-downtime backup/restore strategy; the only documented approach requires stopping all cluster writes during backup
- Characterization: "Unstable local S3 provider" — not a maintainer statement, but a user assessment in context of Sentry 25.10 deployment
- Status: open, unresolved
- Context: Sentry self-hosted uses SeaweedFS as embedded storage; this is a general-purpose app workload, not a lakehouse-specific concern
- Backup limitation is confirmed by the official Data Backup wiki (which also self-admits limited testing scope)
- Note: This issue reflects the experience of an embedded/application-level deployment, not a dedicated infrastructure team managing SeaweedFS as a standalone service — different from an enterprise lakehouse deployment context

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — GitHub issue page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — user-reported issue with specific version context
