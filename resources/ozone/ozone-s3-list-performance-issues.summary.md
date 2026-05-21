---
source: https://github.com/apache/ozone/discussions/7501
component: ozone
type: article
evidence-tier: official
accessible: true
benchmark-age: 2024
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

This GitHub Discussion on the Apache Ozone repository documents severe S3 LIST performance failures in Ozone 1.4.0 on HDD-only clusters, with the issue partially persisting into 1.4.1. The benchmark data dates to approximately 2024 (Ozone 1.4.x era) and predates the 2.0 and 2.1 releases; it may not reflect current behavior but documents a real failure mode that required specific workarounds. This benchmark data is >2 years old relative to the 2026-05-21 research date — treat as stale.

## Key Points

- **LIST failures in 1.4.0**: The S3 benchmark LIST operation generated 23,503 errors and could not complete on full HDD configurations.
- **Small object performance**: Both READ and WRITE performance with small objects was described as "bad even for an object store" on comparable hardware.
- **Version progression**: 1.4.1 fixed some LIST issues on hybrid (HDD+SSD) setups but introduced new WRITE and MIXED failures on pure HDD datanodes.
- **Workarounds required**: Moving Ratis metadata to SSD, increasing heap to 31 GB, and switching to `FILE_SYSTEM_OPTIMIZED` bucket layout stabilized LIST — but degraded READ throughput by ~75%.
- **Benchmark staleness**: Data reflects 1.4.x behavior; significant performance fixes are documented in Ozone 2.0 and 2.1. No independent 2025–2026 benchmark covering the same workloads is available for direct comparison.
- **Key concern for lakehouse**: Iceberg table operations issue many small LIST requests for manifest files; this failure mode is directly relevant to lakehouse workloads.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (GitHub community discussion)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: practitioner benchmark with specific error counts and configuration details — reliable evidence of historical limitations
