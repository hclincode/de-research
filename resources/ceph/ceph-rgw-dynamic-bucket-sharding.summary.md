---
source: https://docs.ceph.com/en/latest/radosgw/dynamicresharding/
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

Official Ceph documentation for RGW Dynamic Bucket Index Resharding. Explains the performance failure mode where large buckets (high object count) cause RGW bucket index RADOS objects to accumulate too many entries, degrading list and write performance. Dynamic resharding automatically detects and corrects this by increasing shard count. Relevant to lakehouse deployments where a single bucket may hold millions of Parquet or Iceberg data files.

## Key Points

- **Problem**: RADOS cannot parallelize concurrent writes to the same object; the bucket index is a RADOS object; high object-count buckets create a write serialization bottleneck
- **Threshold**: Operator should target ≤100,000 entries per shard; prime shard counts distribute entries more evenly (e.g., 7,001 preferred over 7,000)
- **Practical threshold for intervention**: 900,000 objects in a single bucket has been cited as where index sharding becomes critical
- **Dynamic resharding** (enabled by default since Nautilus):
  - Automatically detects when a bucket index shard exceeds the configured threshold
  - Increases shard count transparently
  - Writes to the bucket are blocked briefly during resharding; reads are unaffected
  - Uses a background process; no operator manual intervention required in most cases
- **Lakehouse implication**:
  - Iceberg table stores many small metadata files (manifest lists, manifests, statistics) and potentially millions of Parquet data files in a single "table" bucket or prefix
  - A lakehouse bucket with millions of objects across many tables will trigger dynamic resharding; this is expected behavior but the brief write stalls during resharding may cause Spark/Trino job delays
  - Operator should pre-shard high-churn buckets during initial setup to avoid reactive resharding during production load
- **Manual pre-sharding**: `radosgw-admin bucket reshard` can be used proactively for known large-bucket workloads
- **Red Hat guidance**: For RGW clusters with large buckets, a static shard count of 1,009–8,011 (prime numbers) is recommended at bucket creation rather than relying on dynamic resharding for very large deployments

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official Ceph project documentation
