---
source: https://ozone.apache.org/release-notes/2.1.0/
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

Apache Ozone 2.1.0 (codename "Joshua Tree") was released December 31, 2025, adding 805 new features, improvements, and bug fixes on top of Ozone 2.0. The release strengthens operational maturity with container reconciliation, snapshot scalability, enhanced S3 gateway, and improved observability. Migration from OpenTracing to OpenTelemetry is a significant operational improvement for production clusters.

## Key Points

- **Release date**: December 31, 2025.
- **Container reconciliation**: New protocol resolves mismatched container states and verifies replica integrity — addresses a critical production failure mode.
- **Snapshot scalability Phase 3**: Enhanced scalability for Ozone snapshots; max snapshot limit increased to 10,000; backup SST pruning interval defaulted to 10 minutes.
- **S3 Gateway improvements**: Added presigned URL support (upload + delete), uppercase metadata headers, S3 STANDARD_IA with erasure coding, default HTTPS on 0.0.0.0:9879.
- **Observability**: Migrated to OpenTelemetry for distributed tracing; new StorageVolumeScannerMetrics and VolumeInfoMetrics; Grafana dashboards for RocksDB and deletion progress; Deletion Progress section in OM Web UI.
- **Listener Ozone Managers**: New read-only, non-voting OM nodes that replicate logs to serve read requests — horizontal read scaling.
- **Metadata performance**: Container metadata migrated from Derby DB to OM RocksDB; OzoneManagerLock refactored to hierarchical pool-based locking; Ratis segment size increased 4 MB → 64 MB.
- **Breaking changes**: Hadoop 3.4 now required (3.1.2 deprecated); 13 configuration default changes; Derby-based metadata paths removed.
- **No Trino/Spark-specific integration changes** documented in this release.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache Ozone documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official release notes — authoritative
