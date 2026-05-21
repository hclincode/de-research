---
source: https://polaris.apache.org/blog/2026/04/04/build-a-local-open-data-lakehouse-with-k3d-apache-ozone-apache-polaris-and-trino/
component: trino
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — Apache Polaris official blog; architecture facts are verifiable; Polaris has a stake in the outcome
---

## Summary

Apache Polaris's official blog documents a production-pattern lakehouse using Ozone (S3 gateway) + Apache Polaris (REST catalog) + Trino. This is the most specific published reference architecture for Trino+Ozone on-premise. Published April 2026.

## Key Points

**Trino-to-Ozone connection method**: Native S3 API only (`fs.native-s3.enabled=true`). Trino sends S3 API calls directly to Ozone's S3 gateway at the configured endpoint. OFS (Ozone native filesystem) is not used for Trino.

**Required Ozone S3 configuration for Trino**:
```
s3.endpoint=http://<ozone-s3-gateway>:9878
s3.path-style-access=true       ← required; Ozone does not support virtual-hosted-style URLs
```

**Catalog**: Apache Polaris (incubating, ASF) configured as Iceberg REST Catalog. Trino uses `iceberg.catalog.type=rest` pointing to Polaris's REST endpoint.

**Critical limitation — no STS support**: Ozone has no AWS STS (Security Token Service) endpoint. Polaris normally uses STS to vend short-lived, scoped credentials per table access. With Ozone, this fails — static long-lived credentials must be used instead. This is a **security operational gap** for production enterprise deployments requiring least-privilege storage access.

**Workaround documented**: Set `stsUnavailable: true` in Polaris catalog config for the Ozone storage location. Use static credentials in Trino's catalog config. In a trusted internal network (own data center), this is operationally acceptable but not ideal.

**Architecture pattern confirmed**:
- Spark writes to Iceberg tables → commits metadata to Polaris → data stored in Ozone
- Trino reads from the same Iceberg tables → resolves metadata via Polaris → reads files from Ozone
- Both engines share one catalog and one storage backend with full consistency via Iceberg's snapshot model

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official ASF project blog, technically detailed, credible
