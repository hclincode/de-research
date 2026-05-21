---
source: https://itsfoss.com/news/minio-moves-away-from-open-source/
component: minio
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — It's FOSS, independent open-source press
---

## Summary

It's FOSS, an independent open-source news outlet, documents MinIO's full withdrawal from open-source development. The article covers the timeline from the March 2025 feature stripping through the December 2025 maintenance mode declaration and into 2026 archival. It confirms that MinIO Inc. has effectively abandoned the community edition in favour of its proprietary AIStor subscription product.

## Key Points

- **March 2025**: MinIO removed the web-based admin UI from the community edition — account management, bucket config, lifecycle policies moved to AIStor only.
- **May 2025**: Additional enterprise features stripped; community edition left with a bare-bones object browser.
- **October 2025**: Community Docker images and pre-built binaries ceased being published. Users requiring CVE patches had to build from source.
- **December 3, 2025**: Co-founder Harshavardhana committed README change declaring maintenance mode — no announcement, no roadmap, no transition guidance.
- **February 2026**: Repository status changed to "no longer maintained."
- **AIStor positioning**: Presented as the only actively supported MinIO-codebase product. Requires a commercial subscription.
- **Community forks**: OpenMaxIO attempted to restore the admin console but is effectively dormant as of mid-2025. No viable community fork exists.
- **Recommended alternatives** (Apache 2.0): SeaweedFS, RustFS (alpha), Ceph RGW.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: independent editorial, credible
