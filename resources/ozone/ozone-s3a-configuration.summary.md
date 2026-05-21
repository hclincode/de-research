---
source: https://ozone.apache.org/docs/user-guide/client-interfaces/s3a/
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

This official Ozone documentation page covers how to configure the Hadoop S3A connector to point to an Ozone S3 Gateway, enabling Spark and other Hadoop-ecosystem tools to access Ozone via s3a:// URLs. This is the primary integration path for Spark with Ozone, distinct from the OFS path.

## Key Points

- **Core settings**: Set endpoint to Ozone S3G (e.g., `http://ozone-s3g-host:9878`), set a valid-looking region (`us-east-1`), enable path-style access (`fs.s3a.path.style.access=true`).
- **Recommended compat settings**: Disable bucket existence probing at startup (`fs.s3a.bucket.probe=0`); disable change detection (`fs.s3a.change.detection.mode=none`) — both are needed to avoid false errors with Ozone's slightly non-AWS S3 behavior.
- **Credentials**: Without security — any key/secret pair works. With Kerberos security — obtain via `ozone s3 getsecret`. Configurable via `core-site.xml` or `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` environment variables.
- **No STS details**: Documentation provides no STS endpoint or temporary credential instructions for S3A.
- **HTTPS**: If S3G uses HTTPS with custom CA, JVM truststore must be configured.
- **Versioning / S3 semantics**: Some S3 behaviors differ from AWS S3; object versioning not fully supported.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative
