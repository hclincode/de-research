---
source: https://www.runatlantis.io/
component: nomad
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-26
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Atlantis is an open-source (Apache 2.0) Terraform pull-request automation tool that runs on-premise as a single Go binary or container. It listens for VCS webhook events, runs `terraform plan` on PRs, and requires explicit `atlantis apply` approval before applying. As of 2026 Atlantis natively supports GitHub, GitLab, Bitbucket, and Azure DevOps. Gitea/Forgejo support is partial: a community issue (runatlantis/atlantis#3538) requested it and recent commits added PR-event handling fixes for Gitea/Forgejo, but the support is not documented as first-class. The alternative for a fully self-hosted Gitea + Forgejo Actions pipeline is raw CI: Forgejo Actions can run `terraform plan` on PRs and post the plan output as a PR comment without Atlantis.

## Key Points

- Atlantis is the canonical on-premise GitOps tool for Terraform; runs as a single binary, no cloud dependency.
- Supports GitHub/GitLab/Bitbucket natively; Gitea/Forgejo support exists via community fixes but is not first-class as of 2026.
- Forgejo Actions (Gitea Actions fork): can replicate the Atlantis plan-on-PR workflow via CI pipeline steps without Atlantis; viable but requires more custom scripting.
- For on-premise deployments: Atlantis + self-hosted GitLab CE is the most battle-tested combination. Atlantis + Gitea/Forgejo is functional but test coverage in the project is thin.
- Terraform state locking during Atlantis apply is handled by the configured backend (Consul or PostgreSQL).

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official project documentation
