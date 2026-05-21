---
source: https://blocksandfiles.com/2025/02/02/ai-storage/ + https://www.hpcwire.com/2025/10/16/future-of-storage-hpc-and-ai-storage-by-the-numbers/ + https://www.trustradius.com/compare-products/dell-powerscale-vs-ibm-storage-scale
component: storage
type: article
evidence-tier: independent
accessible: true
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — trade press and analyst sources
---

## Summary

Market-level overview of the enterprise parallel and scale-out storage landscape for HPC and analytics workloads as of 2025–2026. Covers market share, architecture approaches, and recent product developments relevant to Spark deployments.

## Key Points

**Market share (on-premise HPC/AI storage, 2025):**
- Dell Technologies (PowerScale): 22.3%
- IBM (Spectrum Scale / Storage Scale): 19.1%
- NetApp: 8.5%
- WEKA: growing rapidly, specific share not disclosed
- VAST Data: reached $200M ARR Jan 2025, projecting $600M by 2026

**Dell PowerScale (OneFS)**
- Scale-out NFS filer, 4–256 nodes
- Project Lightning (2025): added pNFS-based parallel I/O — directly competes with WEKA/VAST for parallel access workloads
- Strong enterprise support, existing Hadoop/Spark integrations
- Best suited where NFS is already the standard and teams want incremental improvement

**IBM Storage Scale (formerly Spectrum Scale / GPFS)**
- Mature parallel filesystem (17 years), POSIX-compliant
- Deep Hadoop and Spark integrations via HDFS API compatibility layer
- Strong on-premise HPC credibility; more complex licensing
- Most directly comparable to Weka for Spark on-premise

**VAST Data**
- DASE architecture, all-flash
- Multi-protocol (NFS, S3, SMB) from single namespace
- $1.17B commercial agreement with CoreWeave (Nov 2025) — strong AI cloud momentum
- Rated 4.8/5 on Gartner Peer Insights

**WEKA**
- Gartner Customers' Choice 2025 (4.9/5, 119 reviews)
- Software runs on standard hardware; client required on compute nodes
- Primarily targeted at GPU AI training; Spark is a secondary workload
- No public pricing

**Key trend**: All vendors are converging on NVMe-over-Fabrics (NVMe-oF) with RDMA as the standard for next-generation high-performance storage fabrics.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: trade press — reliable market data, minor vendor-influenced framing in individual vendor articles
