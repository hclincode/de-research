---
source: https://docs.cloudera.com/cdp-private-cloud-upgrade/latest/hdfs-ozone-migration/topics/hdfs-ozone-migration-intro.html
component: hdfs
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Cloudera's official CDP Private Cloud migration guide covers the process for migrating data from HDFS to Apache Ozone. Cloudera is a vendor-adjacent source (they sell CDP which includes both HDFS and Ozone, and have a commercial interest in the migration). The guide is technically detailed and largely tool-neutral. It documents the DistCp-based migration procedure including checksum handling, encryption zones, bucket type selection (OBS vs. FSO), and data validation steps. Cloudera's CDP Private Cloud Base supports both HDFS and Ozone for storage.

## Key Points

- Standard migration tool: `hadoop distcp` — copies from HDFS source to Ozone target (or reverse).
- Checksum mismatch: HDFS uses CRC32C; Ozone uses CRC32 — requires `-Ddfs.checksum.combine.mode=COMPOSITE_CRC` for clusters on Hadoop 3.1.1+.
- Encryption zones: if source data is in HDFS encryption zone or Ozone encrypted bucket, checksums will not match — use `-skipcrccheck`.
- Bucket type: OBS (object-store) buckets require `-direct` flag; FSO (file-system optimized) buckets support atomic renames.
- Validation: compare `hdfs du` output on source and target after migration.
- Cloudera CDP supports HDFS and Ozone as co-equal storage options in CDP Private Cloud Base.
- Cloudera HDP support ended; CDH 6.x support ended; CDP 7.x End of Support dates range from Q1 2026 to June 2026 for various versions.
- Migration must be completed before CDP support window closes.
- WanDisco (now IBM LiveData Migrator) provides live migration tooling for zero-downtime HDFS-to-Ozone migration.

## Security Notes

No issues detected. Source is vendor-adjacent (Cloudera); migration procedure is corroborated by official Ozone DistCp docs and Preferred Networks practitioner account.
Checks performed:
- Malicious or obfuscated code: n/a (documentation)
- Suspicious URLs or redirects: none; docs.cloudera.com domain
- Content quality / AI-generated: high; detailed technical procedure documentation
