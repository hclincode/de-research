---
source: https://juicefs.com/docs/community/production_deployment_recommendations/
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

Official JuiceFS Community Edition production deployment recommendations page. Covers metadata engine HA configurations, object storage selection, client configuration, and operational best practices. Provides the authoritative guidance for enterprise-grade on-premise deployments of the Community Edition.

## Key Points

- Redis for production: deploy with Redis Sentinel (minimum 3 Sentinel nodes monitoring a primary + replica pair); AOF persistence enabled; not Redis Cluster (hash slot limitations)
- TiKV for production: 3-node PD cluster + 3-node TiKV cluster minimum; Raft protocol ensures durability; recommended for billion-scale file counts
- PostgreSQL/MySQL for production: configure with read replicas; suitable when durability > performance
- Object storage: use any S3-compatible system (MinIO, Ceph S3 gateway, on-premise S3); must be highly available independently of JuiceFS
- Client cache: configure `--cache-dir` to local fast storage (NVMe SSD); do NOT use --writeback in production (data loss risk)
- GC: schedule `juicefs gc` as a regular cron job; do not rely on manual invocation
- Metadata backup: `juicefs dump` for periodic metadata backups; v1.3+ supports binary backup of 100M files in minutes
- Network: metadata engine should be co-located or on low-latency links with JuiceFS clients; high-latency metadata paths severely impact small-file performance
- POSIX mount vs Hadoop SDK: for Spark/Trino lakehouse workloads, Hadoop Java SDK (jfs://) is recommended over POSIX mount for better integration with Hadoop's data locality and FileSystem API
- Monitoring: JuiceFS exposes Prometheus metrics; operational observability is available

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation
