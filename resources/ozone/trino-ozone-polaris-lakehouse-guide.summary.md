---
source: https://polaris.apache.org/blog/2026/04/04/build-a-local-open-data-lakehouse-with-k3d-apache-ozone-apache-polaris-and-trino/
component: ozone
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

Published April 4, 2026, this Apache Polaris official blog post is the most current authoritative source for Trino+Ozone integration. It walks through a full local deployment of the Trino+Polaris+Ozone Iceberg lakehouse stack on Kubernetes. The article explicitly documents the no-STS-endpoint limitation in Apache Ozone and its workaround, making it the primary source for Question 4 in the research brief.

## Key Points

- **Trino protocol**: Trino uses the S3 Gateway (S3G) — not OFS — to access Ozone data. Path-style access (`s3.path-style-access=true`) is required.
- **Confirmed STS gap**: "Ozone has no STS endpoint currently, so credential vending doesn't work here." Workaround: set `stsUnavailable: true` in the Polaris catalog and pass static credentials directly to Trino via `s3.aws-access-key` / `s3.aws-secret-key`.
- **Native-S3 filesystem**: Trino uses `fs.native-s3.enabled=true` with these config properties: `s3.endpoint`, `s3.path-style-access=true`, `s3.aws-access-key`, `s3.aws-secret-key`, `s3.region=us-east-1` (any valid-looking region accepted).
- **Credential security**: In non-secure Ozone mode, S3G accepts any access/secret key pair with no validation — suitable for dev/test only.
- **Direct data I/O**: Trino reads and writes Parquet files directly to Ozone using static S3 credentials; Polaris only handles metadata.
- **Resource requirements**: Ozone is described as "resource-hungry"; minimum 8 GB RAM / 4 CPUs recommended.
- **Bucket protocol**: Catalog base location uses `s3://` (not `s3a://`) with Polaris; Trino's native-S3 driver handles the translation.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official ASF Polaris blog)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: technical walkthrough with reproducible configuration — high quality
