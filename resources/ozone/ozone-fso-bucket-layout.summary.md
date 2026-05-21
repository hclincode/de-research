---
source: https://ozone.apache.org/docs/core-concepts/namespace/buckets/layouts/file-system-optimized/
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

The File System Optimized (FSO) bucket layout is the default in Apache Ozone for Hadoop-compatible filesystem workloads. It stores intermediate directories in a separate DirectoryTable and references children by parent unique ID, enabling O(1) directory rename and prefix-based deletion — critical for Iceberg's frequent metadata operations. FSO is strongly recommended for lakehouse deployments.

## Key Points

- **Default layout**: FSO is the default bucket layout in Ozone 2.x for filesystem-oriented buckets.
- **O(1) rename**: Directory rename updates only a metadata pointer; all child files (millions of them) stay in place — avoids the O(n) scan required by Object Store (OBS) layout.
- **Efficient deletion**: Uses prefix-based queries on shared parent IDs — no full namespace scan needed.
- **Iceberg relevance**: Iceberg relies on atomic metadata updates; FSO's atomic rename semantics support this. Frequent manifest and snapshot file creation benefits from efficient small-file metadata handling.
- **Recommended for**: Hadoop FS interface workloads, analytics (Hive, Spark), hierarchical directory structures, trash/recycle bin functionality.
- **Key limitation**: FSO buckets cannot be used as pure object-store buckets; keys cannot be created at the volume root level.
- **S3 LIST performance**: FSO layout (rather than OBS) is one of the workarounds cited for LIST performance issues on small-file workloads (see discussion #7501).

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative
