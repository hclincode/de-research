---
source: https://docs.weka.io/weka-system-overview/weka-client-and-mount-modes
component: weka
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — official product documentation
---

## Summary

Official Weka documentation describing client installation requirements and mount modes. The Weka client is a POSIX-compliant filesystem driver that must be installed on every compute node that needs POSIX/WekaFS access. For S3-based access (Trino S3 connector, Spark with S3A), no Weka client is needed on compute nodes — only the S3 gateway servers require the Weka software. This is a critical operational distinction for lakehouse deployments: S3-mode access decouples compute nodes from the proprietary Weka client dependency; POSIX-mode access does not.

## Key Points

- **POSIX/WekaFS access**: Requires the Weka client driver installed on every application server (Spark executor, Trino worker). The client is a kernel module or FUSE layer — it is a mandatory OS-level installation.
- **S3 access mode**: Compute nodes do NOT need the Weka client installed. They communicate with the Weka S3 gateway over the network using standard S3 API calls. This enables "clientless" compute nodes for analytics workloads.
- **Minimum cluster for on-premise**: 8 servers minimum for production on-premise bare-metal deployments (6 for cloud). This sets a significant minimum procurement threshold.
- **NFS access**: Weka also supports NFS (v3 and v4) as an additional protocol, which like S3 does not require the client on compute nodes.
- **Client installation footprint**: The Weka client adds OS-level dependencies (kernel modules or FUSE). This complicates Kubernetes deployments, OS upgrades, and node provisioning automation — a known operational overhead.
- **Mount modes**: Clients can mount in "stateless" mode (Weka manages data path) or "direct" mode; both require the client package. Stateless mode simplifies some operational aspects.

## Security Notes

No issues detected. Official vendor documentation.
Checks performed:
- Malicious or obfuscated code: n/a
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — official technical documentation
