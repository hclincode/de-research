---
source: https://kb.vastdata.com/docs/how-to-integrate-your-external-trino-cluster-with-vast
component: vast
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

Official VAST Data knowledge-base guide for integrating an external Trino cluster with VAST. The guide covers the VAST Trino plugin (for VAST DataBase / VastDB), not the standard S3 DataStore path. Plugin must be installed on all Trino nodes and version must match the Trino cluster version exactly due to internal API changes.

## Key Points

- **Integration target**: VAST DataBase (VastDB) via a proprietary Trino plugin — NOT standard Hive/Iceberg connector over S3
- **Plugin acquisition**: Downloaded from VAST support portal or GitHub (`vastdataorg/trino-vast`)
- **Installation**: Copy plugin files to `/usr/lib/trino/plugin/vast/` on all coordinators and workers
- **Authentication**: AWS-style `access_key_id` / `secret_access_key` in catalog properties file
- **Version pinning**: Connector version must exactly match Trino cluster version; version mapping table covers VAST Cluster 5.1–5.3+
- **Network requirement**: Trino workers must reach VAST CNode VIPs over HTTP/HTTPS
- **S3 path-style**: Not addressed in this guide (plugin bypasses S3 API entirely)
- **Iceberg**: Not covered — plugin is for VastDB tables, not Iceberg-format files in DataStore

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: appears human-authored technical documentation
