---
source: https://github.com/minio/minio/discussions/21296
component: minio
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official MinIO GitHub discussion thread
---

## Summary

An official MinIO GitHub discussion thread on AGPLv3 licensing clarifies the copyleft reach for enterprise users who access MinIO over the network. MinIO's own position is that the AGPL's copyleft provisions do not extend to applications that communicate with the MinIO server using standard network interfaces (HTTP/S3 API). However, enterprise legal teams routinely apply a more conservative reading, and the AGPL still requires organisations to publish source code if they ever modify MinIO and provide it as a service — a compliance burden that many enterprise legal departments consider unacceptable.

## Key Points

- **AGPL v3 adopted**: MinIO moved from Apache 2.0 to AGPLv3 in 2021. The OSS repository was AGPLv3 until it was archived.
- **Network copyleft**: AGPLv3 adds a network use clause — if you modify MinIO and run it as a service (even internally), you must make your modifications available.
- **Client application reach**: MinIO's interpretation: client applications connecting over standard S3 API are not derivative works. However, legal guidance varies and many enterprise legal teams reject AGPL dependencies categorically.
- **Enterprise legal position**: Many enterprises (financial services, healthcare, defence) maintain internal policies prohibiting AGPL software without a commercial license waiver, regardless of how the software is technically used.
- **Current relevance**: Moot for MinIO CE as the repository is archived. Relevant for organisations evaluating whether to use a pinned CE release — they cannot obtain security patches without a commercial AIStor subscription.
- **AIStor license**: Proprietary subscription agreement (not AGPL). Eliminates the copyleft risk but introduces commercial dependency.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — GitHub discussion thread
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official project thread, credible
