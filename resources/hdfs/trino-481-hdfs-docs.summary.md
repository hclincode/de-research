---
source: https://trino.io/docs/current/object-storage/file-system-hdfs.html
component: trino
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

The Trino 481 (current as of May 2026) official documentation page for HDFS file system support confirms that HDFS access via `fs.hadoop.enabled=true` is still a supported, non-deprecated configuration path. The page covers both HDFS 2.x and 3.x and details authentication options (NONE and Kerberos). Importantly, the page makes clear that `fs.hadoop.enabled` should be used "only for HDFS" — the legacy S3/Azure/GCS implementations that previously also used this flag have been removed in Trino 481. HDFS itself retains active documentation and is not marked deprecated.

## Key Points

- Trino 481 (May 2026) still has an active, non-deprecated HDFS documentation page.
- `fs.hadoop.enabled=true` must be set in the catalog configuration file to enable HDFS access.
- HDFS 2.x and 3.x are both supported.
- Guidance explicitly states: "Use `fs.hadoop.enabled` only for HDFS" — legacy S3/Azure/GCS via this flag has been removed.
- Authentication: supports `NONE` (simple Hadoop auth) and `KERBEROS` with optional user impersonation.
- Wire encryption is supported but "may impact query execution performance."
- Keytab files require strict access controls (standard Kerberos security requirement).
- No removal date or deprecation warning shown in the documentation as of Trino 481.
- Distinction is important: `fs.hadoop.enabled` for cloud storage was removed; `fs.hadoop.enabled` for actual HDFS was preserved.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (documentation)
- Suspicious URLs or redirects: none; official trino.io domain
- Content quality / AI-generated: high; official Trino project documentation
