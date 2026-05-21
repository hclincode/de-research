---
source: https://ceph.io/en/news/blog/2025/v20-2-0-tentacle-released/
component: ceph
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

Ceph Tentacle (v20.2.0) is the 20th stable release of Ceph, released November 2025. It is the current stable release as of May 2026. Key highlights relevant to lakehouse deployments include FastEC (long-anticipated erasure coding performance improvements delivering partial reads and partial writes), RGW bucket listing and scrub speedups via faster OMAP iteration, S3 `GetObjectAttributes` support, and a change of the default EC plugin from Jerasure to ISA-L for improved performance on modern CPUs.

## Key Points

- **Release date**: November 2025 (v20.2.0); current stable as of May 2026
- **FastEC**: New erasure coding I/O path with partial reads and partial writes; primary target is RBD (block) and CephFS (file) but benefits RGW for smaller objects; 2–3x erasure coding performance improvement cited
- **Default EC plugin changed**: Jerasure → ISA-L (Intel ISA-L library for hardware-accelerated coding)
- **RGW improvements**:
  - All components switched to faster OMAP iteration interface → faster bucket listing and scrub
  - `GetObjectAttributes` S3 API added
  - `LastModified` timestamps now truncated to the second for AWS S3 compatibility
  - Multi-site automation, tiering, policies, lifecycle, notifications, and granular replication enhancements
  - OAuth 2.0 integration
- **BlueStore improvements**: improved compression, new faster WAL (write-ahead-log)
- **NVMe/TCP gateway groups**: multi-namespace support
- **Erasure coding caveat**: FastEC partial-read benefit for object storage depends on access pattern; RGW still issues stripe-width reads for partial object reads in EC pools in the general case; FastEC closes but does not eliminate the replication vs. EC small-read gap

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph Foundation blog post, release announcement
