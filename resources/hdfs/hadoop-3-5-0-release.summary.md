---
source: https://hadoop.apache.org/release/3.5.0.html
component: hdfs
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

Apache Hadoop 3.5.0 was released on April 2, 2026, containing 485 bug fixes, improvements, and enhancements since the 3.4 line. This is the first stable release of the 3.5 line and is the first Hadoop release requiring Java 17 on the server side (Java 17 and 21 supported on client side). Active development continues with Hadoop 3.6.0-SNAPSHOT already in progress. Hadoop 3.4.3 was also released on February 24, 2026 as a maintenance release.

## Key Points

- Hadoop 3.5.0 released April 2, 2026; Hadoop 3.4.3 released February 24, 2026 — both active in 2026.
- Java 17 is now required on the server side; Java 21 also supported on the client side.
- HDFS gains finer-grained locking on concurrent NameNode operations (scalability improvement).
- HDFS Router-based Federation (RBF) gains asynchronous RPC processing for improved throughput.
- RBF now supports storing delegation tokens on MySQL (improvement over Zookeeper for token operations).
- 3.6.0-SNAPSHOT is the next development target (last publish May 15, 2026).
- Apache License 2.0 — fully open source.
- Multiple active release lines: 3.5.x and 3.4.x are both supported.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (release announcement, no executable content)
- Suspicious URLs or redirects: none; official ASF domain
- Content quality / AI-generated: high; official Apache Software Foundation release page
