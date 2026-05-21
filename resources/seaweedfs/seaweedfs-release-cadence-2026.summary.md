---
source: https://github.com/seaweedfs/seaweedfs/releases
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

SeaweedFS maintains an aggressive release cadence in 2026, with versions 4.19 through 4.27 released between April 8 and May 20, 2026 — approximately 2-3 releases per week. Recent releases address EC shard reliability, FUSE mount fixes, S3 IAM enhancements, Iceberg catalog improvements, NFS server additions, and POSIX compliance. Version 4.23 was explicitly marked as unsafe for erasure coding on multi-disk servers; 4.24 corrected this — demonstrating the risk of rapid cadence in production environments that upgrade frequently.

## Key Points

- Release cadence: 2-3 releases per week in May 2026; ~9 releases in 6 weeks (4.19 to 4.27)
- Total releases: 301 as of May 2026
- Recent fix categories: EC shard recovery (4.25, 4.26), S3 IAM (4.20, 4.21, 4.27), Iceberg catalog (4.19, 4.23), FUSE mount (4.21, 4.27), NFS server (4.21)
- Critical regression: version 4.23 marked "unsafe for erasure coding with multi-disk servers" — rolled back/fixed in 4.24; illustrates that rapid cadence introduces production upgrade risk
- Positive signal: maintainer responsiveness — critical EC regression identified and fixed within days
- Implication for enterprise: pin to a tested version; avoid tracking latest on production; establish internal validation before upgrades
- Latest stable at research date: 4.27 (May 20, 2026)

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official GitHub releases page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official release records
