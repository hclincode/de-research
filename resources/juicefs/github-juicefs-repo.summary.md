---
source: https://github.com/juicedata/juicefs
component: juicefs
type: github-repo
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

JuiceFS is an Apache 2.0 licensed distributed POSIX file system written in Go (84%) with a Java SDK (10.5% for Hadoop ecosystem integration). The latest stable release is v1.3.1 (December 2, 2025). The project has 13.6k GitHub stars, 1.2k forks, and 147 open issues as of the research date. It provides full POSIX and Hadoop-compatible interfaces, an S3-compatible gateway, and Kubernetes CSI driver support. The README states it passed all 8,813 POSIX compatibility tests (pjdfstest suite).

## Key Points

- License: Apache License 2.0 (Community Edition)
- Latest stable: v1.3.1, released December 2, 2025
- Languages: Go (84.3%), Java (10.5%), Shell, Python, C
- Default chunk size: 64 MiB; default block size: 4 MiB
- Provides POSIX, HDFS (Hadoop Java SDK), S3 gateway, and WebDAV interfaces
- Passed 8,813 POSIX compatibility tests (pjdfstest)
- Supports global file locking (BSD and POSIX locks)
- Close-to-open consistency guarantee by default
- Atomic metadata operations
- LZ4/Zstandard compression support
- Multi-client shared access with strong (close-to-open) consistency
- Data encryption in transit and at rest
- Enterprise Edition is separate, proprietary, and not open-source

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Not applicable — official project repository
- Suspicious URLs or redirects: None detected
- Content quality / AI-generated: High quality; official project source
