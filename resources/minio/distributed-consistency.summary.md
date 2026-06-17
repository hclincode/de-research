---
source: https://github.com/minio/minio/blob/master/docs/distributed/README.md
component: minio
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-06-17
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

MinIO's own distributed-mode documentation states it provides **strict read-after-write and
list-after-write consistency** for all I/O in both distributed and standalone modes — i.e. a single
MinIO deployment is strongly consistent, matching AWS S3's model. Crucially, this guarantee is
conditional on the underlying disk filesystem and is **not** preserved across separate deployments.

## Key Points

- "MinIO follows strict **read-after-write** and **list-after-write** consistency model for all i/o
  operations both in distributed and standalone modes."
- The guarantee holds **only on filesystems with correct semantics**: "only guaranteed if you use disk
  filesystems such as xfs, zfs or btrfs". **ext4 is explicitly unsafe** — "ext4 does not honor POSIX
  O_DIRECT/Fdatasync semantics, ext4 trades performance for consistency guarantees."
- **NFS voids the guarantee**: "If MinIO distributed setup is using NFS volumes underneath it is not
  guaranteed MinIO will provide these consistency guarantees" (NFSv4 preferred over NFSv3 if NFS is
  unavoidable, but still weaker than local FS).
- Implication for the OpenResty→HAProxy→MinIO investigation: MinIO itself is strongly consistent
  **within one deployment on a proper filesystem**; inconsistency must therefore come from the
  architecture around it (independent/replicated backends, proxy retries) or from a wrong filesystem.

## Security Notes

No issues detected. Official project documentation.
Checks performed:
- Malicious or obfuscated code: n/a (documentation).
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; primary project doc.
