---
source: https://github.com/trinodb/trino/discussions/18026
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

This Trino GitHub Discussion documents concrete JAR-level integration failures when attempting to use Apache Ozone's native filesystem with Trino's Iceberg connector. The root cause is a Protobuf class conflict between Ozone's client library and Trino's internal dependencies. Two viable paths exist: use S3G, or apply Protobuf shadowing.

## Key Points

- **Failure mode**: Adding `ozone-filesystem-hadoop3-1.3.0.jar` to Trino's Iceberg plugin directory causes `java.lang.NoClassDefFoundError: com/google/protobuf/ServiceException`, then after adding protobuf JAR, `IllegalStateException` from RPC client initialization with 500+ failover retries.
- **Root cause**: Ozone's client library requires Protobuf shadowed to `io.trino.hadoop.$internal` namespace; conflict with Trino's bundled Protobuf version.
- **Fix**: Apache Ozone PR #7729 implements Protobuf shadowing; shaded JARs placed in `plugin/iceberg/hdfs` or `plugin/hive/hdfs`.
- **Practical outcome**: Most practitioners shifted to the S3G gateway to avoid JAR conflicts entirely.
- **Configuration used**: `connector.name=iceberg`, `hive.metastore.uri=thrift://...`, `hive.config.resources=...ozone-site.xml` — this configuration led to the failures.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (GitHub community discussion on Trino project)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: practitioner troubleshooting with specific error messages — high reliability for documenting known issues
