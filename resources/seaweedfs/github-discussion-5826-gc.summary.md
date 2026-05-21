---
source: https://github.com/seaweedfs/seaweedfs/discussions/5826
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

GitHub Discussion #5826 documents a case where automatic garbage collection ran but did not reclaim disk space (60 GB consumed for 17 files ~180 MB each). Investigation revealed multiple issues: the default GC threshold was too high, there were orphaned filer entries, and different master nodes showed different file listings (filer consistency bug). Data corruption was also discovered (truncated/half-size files). The user resolved disk space by lowering the vacuum threshold and running `volume.fsck -reallyDeleteFromVolume`.

## Key Points

- Problem: auto GC ran but disk not reclaimed; 60 GB for 17 ~180 MB files
- Root cause: default 30% threshold too conservative; vacuum ran but no compaction triggered
- Resolution: run `volume.vacuum -garbageThreshold=0.2` to force compaction + `volume.fsck -reallyDeleteFromVolume` for orphans; disk dropped from 72% to 4% (56.2 GB → 3.3 GB)
- Secondary finding: different master nodes showed different file listings via `weed shell` — filer state divergence confirmed
- Tertiary finding: some uploaded files were truncated (half-size) and returned corrupted content on download — data corruption discovered during investigation
- Ghost bucket: deleted bucket (`s2-backup`) temporarily reappeared in listings
- Maintainer recommendation: use single filer or shared filer store to address synchronization problems; update to latest version before production use
- Operational implication: monitoring threshold-based compaction rather than relying on automatic vacuum is necessary; default GC settings are insufficient for many real-world delete patterns

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — GitHub discussion page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — user-reported operational experience with specific numbers
