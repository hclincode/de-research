---
source: https://www.min.io/pricing
component: minio
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official vendor pricing page
---

## Summary

MinIO's official pricing page describes the AIStor tiered subscription model introduced in December 2025. AIStor Free is a single-node-only, non-clustered deployment licensed under a proprietary MinIO AIStor Free Tier License Agreement effective December 22, 2025. It is not open-source and does not qualify under any OSI-approved license. AIStor Enterprise pricing is approximately $0.02/GB/month ($20,480/PB/month or roughly $245,760/PB/year at list price), with volume discounts available above 1 PB.

## Key Points

- **AIStor Free Tier**: Single-node only. No distributed clustering, no high availability. Licensed under a proprietary "AIStor Free Tier License Agreement" — not FOSS. Suitable for development, homelab, research.
- **AIStor Enterprise Lite**: Entry-level commercial tier. Minimum ~$96k/year.
- **AIStor Enterprise**: Full distributed HA. ~$0.02/GB/month list price (~$245,760/PB/year). Volume discounts apply north of 1 PB.
- **Critical on-premise enterprise constraint**: AIStor Free's single-node restriction makes it ineligible for any production HA lakehouse. AIStor Enterprise requires a commercial subscription, violating the open-source constraint.
- **Support model**: AIStor Enterprise includes the MinIO Subscription Network (SUBNET) for production SLA/SLO-backed support.
- **License divergence**: AIStor is governed by a proprietary agreement, not AGPLv3. The old Community Edition was AGPLv3 but is now archived.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none — vendor pricing page
- Suspicious URLs or redirects: none
- Content quality / AI-generated: standard vendor marketing copy
