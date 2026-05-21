---
source: https://blocksandfiles.com/2023/03/26/we-object-minio-says-no-more-open-license-for-you-weka/
component: weka
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — trade press (Blocks & Files); corroborated by TechTarget and MinIO's own blog
---

## Summary

In March 2023, MinIO publicly accused Weka of violating its open-source license by distributing the MinIO binary — including the server, client, and WARP benchmarking tool — inside Weka's product without proper attribution. MinIO revoked Weka's license to all MinIO software effective immediately. Weka contested this, claiming it used only Apache 2.0-licensed MinIO code (not AGPL v3 code), that its Apache 2.0 licenses are irrevocable, and that it was fully compliant with attribution requirements. This dispute has direct relevance to the Weka S3 gateway: Weka's S3 protocol layer was built on or adjacent to MinIO's S3 implementation. Weka subsequently built its own S3 implementation to reduce this dependency. The dispute was covered by Blocks & Files, TechTarget, and Hacker News (item 35299665).

## Key Points

- **Root of dispute**: Weka embedded MinIO binaries (Apache 2.0 AND AGPL v3 code, per MinIO) inside its product without source disclosure, violating AGPL requirements.
- **Weka's position**: Only Apache 2.0-licensed MinIO code was used; AGPL code was never incorporated; Apache 2.0 licenses are irrevocable by definition.
- **MinIO's action**: Revoked all MinIO licenses (Apache 2.0 and AGPL v3) for Weka, citing the right to terminate under the license terms.
- **Relevance to Weka S3**: This dispute reveals that Weka's S3 gateway had historical roots in MinIO's codebase. Weka subsequently developed its own S3 implementation, which is what appears in current documentation.
- **Risk signal**: This incident is a caution flag for enterprises evaluating Weka's open-source licensing hygiene. The dispute was never publicly resolved through a court ruling — it ended with Weka asserting compliance and building its own S3 implementation.
- **Practitioner reaction (HN)**: Hacker News discussion (item 35299665) showed skepticism toward Weka's defense, with practitioners noting that distributing AGPL-licensed binaries without source disclosure is a common but serious compliance failure. Some comments noted difficulty evaluating Weka's S3 performance claims given the embedded third-party codebase.

## Security Notes

No issues detected. Trade press article covering a documented licensing dispute.
Checks performed:
- Malicious or obfuscated code: n/a
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — event-driven reporting with named parties and public documentation
