---
source: https://ozone.apache.org/docs/2.0.0/interface/ofs.html
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

This official Ozone 2.0 documentation page describes the OFS (Ozone File System) interface using the `ofs://` scheme. OFS provides a full Hadoop Compatible FileSystem view spanning all volumes and buckets, unlike the legacy `o3fs://` which is scoped to a single bucket. Spark, Hive, Flink, and MapReduce use OFS natively; Trino requires additional work.

## Key Points

- **OFS vs o3fs**: OFS supports operations across all volumes and buckets with a hierarchical view from root; o3fs is scoped to a single bucket. OFS is the recommended scheme for new deployments.
- **Path format**: `ofs://omservice/volume1/bucket1/dir1/key1` — OM host or service ID in the authority component.
- **Temp directory**: `/tmp` is a special mount point; configurable as per-user or shared (`ozone.om.enable.ofs.shared.tmp.dir`).
- **Trash**: Deleted files go to bucket-specific trash; configured via `fs.trash.classname` and `fs.trash.interval`.
- **Recursive operations**: Supports recursive listing across volumes/buckets/keys with ACL-based filtering.
- **Limitations**: Cannot create keys directly under root or volumes (returns error). Bucket and volume names cannot contain underscores or other unsupported characters.
- **Trino compatibility**: Not documented — open-source Trino does not natively support `ofs://`; see discussions #5004 and Trino #18026.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative
