---
source: https://github.com/juicedata/juicefs/discussions/2700
component: juicefs
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

GitHub community discussion reporting data consistency issues with JuiceFS v1.0.0 in a Kubernetes deployment. A user reported unexpected metadata store growth (from 18,250 to 147,859 entries overnight) and "file does not exist" errors during filesystem checks (juicefs fsck). The issues were reported in a production-like Kubernetes environment. This represents an early production experience; v1.0.0 was released in 2022 and significant improvements have been made through v1.3.x by 2025.

## Key Points

- Issue: Metadata store grew unexpectedly (18,250 → 147,859 entries overnight) in a Kubernetes JuiceFS v1.0.0 deployment
- "File does not exist" errors appeared during `juicefs fsck` runs, suggesting metadata/object storage inconsistency
- Version: JuiceFS v1.0.0 (initial 1.x release, 2022); current stable is v1.3.1 (December 2025)
- The metadata growth issue may be related to GC not running frequently enough, leaving orphaned metadata
- JuiceFS garbage collection must be run manually (`juicefs gc`) or scheduled separately — it does not run automatically in the background by default
- This is a community-reported limitation: GC does not run automatically, requiring operational discipline
- Close-to-open consistency does not prevent metadata growth from orphaned objects if GC is not scheduled
- Hacker News community expressed concern about: (1) Redis as a single point of failure; (2) added operational complexity of maintaining a separate metadata service alongside object storage

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: GitHub discussion; no executable content
- Suspicious URLs or redirects: None
- Content quality / AI-generated: Community discussion; human-authored
