---
source: https://ozone.apache.org/blog/2026/01/29/disk-balancer-preview
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

Published January 29, 2026, this Apache Ozone blog post previews the Disk Balancer feature shipping in Ozone 2.2. The feature addresses intra-node disk imbalances that persist even after cluster-wide balancing, a known operational failure mode in production clusters where disks fill unevenly after adding or replacing hardware.

## Key Points

- **Problem**: Even cluster-balanced nodes can have individual disks at 95% while others are idle — creates I/O hotspots and increases failure risk.
- **Mechanism**: Uses Volume Data Density metric (deviation from node average). When deviation exceeds configurable threshold, balancing begins.
- **Container migration**: Copy-Validate-Replace — copy to destination, transition to RECOVERING state, delete original only after confirmed successful import.
- **Only closed containers**: Open (actively writing) containers are not moved; only closed containers eligible.
- **Ships in Ozone 2.2**: Not yet available in 2.0.0 or 2.1.0.
- **Configuration**: `hdds.datanode.disk.balancer.enabled=true`. Defaults: 10% threshold, 10 MB/s bandwidth, 5 parallel threads, auto-stop when balanced.
- **CLI support**: Start, stop, status, report, configuration update commands targeting individual or all IN_SERVICE datanodes.
- **Operational benefit**: Reduces manual disk management; particularly valuable during hardware replacement in large clusters.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache Ozone blog)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official blog post with technical detail — high quality
