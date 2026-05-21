---
source: https://juicefs.com/docs/community/hadoop_java_sdk/
component: juicefs
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

Official JuiceFS documentation for integrating JuiceFS with the Hadoop ecosystem via the JuiceFS Hadoop Java SDK. The SDK implements the `org.apache.hadoop.fs.FileSystem` interface, making JuiceFS a drop-in replacement for HDFS in Spark, Hive, Trino (legacy Hadoop connector), and other Hadoop-compatible engines. JuiceFS is accessed via the `jfs://` URI scheme. Configuration is placed in `core-site.xml` or passed as Spark submit arguments.

## Key Points

- SDK extends `org.apache.hadoop.fs.FileSystem`; compatible with Hadoop 2.x and 3.x
- URI scheme: `jfs://<volume-name>/path`
- Configuration in core-site.xml:
  - `fs.jfs.impl=com.juicefs.JuiceFileSystem`
  - `fs.AbstractFileSystem.jfs.impl=com.juicefs.JuiceFS`
  - `juicefs.meta=<metadata-engine-URL>`
- Or pass as Spark CLI args: `--conf spark.hadoop.fs.jfs.impl=com.juicefs.JuiceFileSystem`
- For data locality: configure `juicefs.discover-nodes-url` to let JuiceFS obtain YARN/Spark node list for BlockLocation computation
- Distributed cache (intra-cluster peer cache): use `juicefs.cache-group` to share cache across nodes in the same group
- Cache directory: set `juicefs.cache-dir` to a local NVMe path in production; adjust `juicefs.cache-size`
- JVM off-heap memory: SDK may require `4 * juicefs.memory-size`; recommend ≥1.2 GB off-heap for compute tasks
- Multiple JuiceFS volumes supported simultaneously by using volume-name-specific config keys (e.g., `jfs1.*`, `jfs2.*`)
- Hadoop SDK JAR must be placed in Spark's classpath

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation
