---
source: https://github.com/trinodb/trino/issues/24878
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

GitHub issue #24878 in the trinodb/trino repository tracks the complete removal of legacy file system support. The issue was opened following the Trino 470 deprecation (February 2025). The broader decoupling effort is tracked in issue #15921. Planned removal was initially targeted for "weeks or months" after February 2025, with one source mentioning a possible March 2025 target tied to a Java 24 upgrade. As of Trino 481 (May 2026), the legacy S3/Azure/GCS code has been confirmed removed, while the HDFS path (`fs.hadoop.enabled` specifically for HDFS) remains. The issue is the authoritative tracking point for HDFS deprecation risk.

## Key Points

- Issue #24878 is the tracking issue for removing the legacy Hadoop FS implementation from Trino.
- Parent tracking issue: #15921 ("Decouple Trino from Hadoop and Hive codebases").
- Deprecated legacy support covered cloud object stores (S3, Azure, GCS) accessed via Hadoop FS shims — not HDFS itself.
- Legacy S3 support confirmed removed in Trino 481: "legacy object storage support through fs.hadoop.enabled and deprecated hive.* properties is no longer available" — this refers to S3/Azure/GCS, not HDFS.
- The HDFS-specific use of `fs.hadoop.enabled` is retained as a separate supported path.
- Related PR #23343 ("Disable legacy filesystem implementation by default") was a precursor.
- No date has been publicly committed for removing actual HDFS support; it is not tracked in this issue.
- Risk: future removal of Hadoop FS libraries could eventually make HDFS access impossible without refactoring; no timeline confirmed.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (GitHub issue tracker)
- Suspicious URLs or redirects: none; github.com domain
- Content quality / AI-generated: high; official project issue tracker
