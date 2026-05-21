---
source: https://ozone.apache.org/roadmap/
component: ozone
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

The official Apache Ozone roadmap (retrieved May 2026) lists planned releases 2.2.0 (Katmai), 3.0.0, and the completed 2.1.0 and 2.0.0. STS temporary credentials (HDDS-13323) are explicitly planned for 2.2.0. The roadmap does not include Trino-specific integration work, small file optimization, or dedicated DR features — these are tracked in JIRA separately.

## Key Points

- **2.2.0 (Katmai) — next minor release**:
  - Disk Balancer (HDDS-5713): intra-node disk rebalancing — ships in 2.2.
  - S3 Lifecycle Configurations — object expiration (HDDS-8342).
  - **STS — temporary, limited-privilege credentials service (HDDS-13323)**: explicitly planned for 2.2.0. Active development confirmed via JIRA updates in November 2025 (HDDS-13926: STS Part 3, IAM policy → OzoneObj/Acl conversion).
- **3.0.0**: Leader Execution at Leader (HDDS-11415) — distributed execution optimization.
- **Not in roadmap**: Trino integration, small file optimization, cross-datacenter DR, enhanced monitoring beyond current Grafana dashboards, multi-tenancy improvements.
- **STS status**: As of May 2026, STS is in development (2.2.0 target) but not yet released; workaround remains static credentials.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache Ozone roadmap page)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official roadmap — authoritative
