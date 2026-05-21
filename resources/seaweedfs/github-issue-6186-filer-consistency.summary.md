---
source: https://github.com/seaweedfs/seaweedfs/issues/6186
component: seaweedfs
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

GitHub Issue #6186, opened November 2024 for SeaweedFS 3.67, documents that multi-filer deployments using local-only filer stores (leveldb2) diverge permanently after network partitions that affect deletion operations. When a deletion's metadata broadcast fails during a partition, other filers continue to serve the file indefinitely even after network recovery. The issue was closed as "not planned" with no fix or mitigation from maintainers.

## Key Points

- Affected configuration: 1 master, 3 volume servers, 3 filer servers using leveldb2 (local storage backend)
- Root cause: local metadata updated first during deletion; if broadcast fails, peer filers retain stale state indefinitely
- Persistence: metadata inconsistency does not self-heal after network recovery
- Maintainer response: issue closed as "not planned" — no fix, no workaround provided by project
- Official mitigation (community-identified): (1) use shared distributed filer store (Redis, Cassandra, PostgreSQL) instead of leveldb2; or (2) consolidate to single-filer deployment
- Risk for lakehouse: Iceberg table snapshots and manifest deletions (during table maintenance) could leave ghost entries visible on some filers in multi-filer configurations with local filer stores
- Implication: multi-filer + local filer store is not safe for environments with frequent deletes and any possibility of network instability

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — GitHub issue page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — tracked GitHub issue with specific reproduction steps
