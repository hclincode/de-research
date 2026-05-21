---
source: https://docs.softwareheritage.org/sysadm/mirror-operations/seaweedfs.html
component: seaweedfs
type: article
evidence-tier: press
accessible: true
benchmark-age: 2023
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Software Heritage — a nonprofit digital archive preserving open source code — documents their production SeaweedFS deployment for mirror operations. Their cluster runs 4 bare-metal machines hosting all SeaweedFS components (3 masters, 1 filer, 4 volume servers), achieving 2,000 objects/second at approximately 200 MB/s sustained throughput for content replication. The filer index for the entire Software Heritage archive required ~2 TB of LevelDB storage as of March 2023.

## Key Points

- Organization: Software Heritage (nonprofit code archive; independent of SeaweedFS project and vendors)
- Scale: full Software Heritage objstorage archive; 2 TB filer index as of March 2023
- Throughput achieved: 2,000 objects/s at ~200 MB/s sustained (content replication process)
- Hardware: 4 bare-metal machines hosting 3 masters, 1 filer, 4 volume servers
- Filer store: LevelDB (single-filer limitation applies; shared filer store recommended for multi-filer)
- Technical finding: standard Redis filer store (redis2) cannot handle all Software Heritage content entries in a single flat directory; the project created a `redis3` backend to circumvent this — a non-trivial operational workaround
- Recommendation from docs: use shared filer store (Redis = best performance for replication); single-filer LevelDB has limitations at archive scale
- Benchmark age: 2023 for filer index size; throughput likely measured at similar time — both are stale relative to 2026 research date but confirm production viability at scale
- This is an independent production deployment with no vendor relationship to SeaweedFS

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official Software Heritage documentation site
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — technical operations documentation with specific numbers
