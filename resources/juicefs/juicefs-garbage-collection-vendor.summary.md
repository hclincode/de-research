---
source: https://juicefs.com/en/blog/engineering/juicefs-garbage-collection
component: juicefs
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

JuiceData blog post (vendor) describing JuiceFS's garbage collection mechanism. JuiceFS uses a copy-on-write model: overwrites and deletions mark old blocks as stale rather than immediately removing them. GC is a separate manual or scheduled process that reconciles metadata with object storage contents and deletes orphaned blocks. Unlike traditional filesystems, GC does not run continuously in the background by default.

## Key Points

- Copy-on-write (COW) model: on overwrite, new data block uploaded, old block marked stale in metadata; old block NOT immediately deleted
- GC process (`juicefs gc`): scans metadata against object storage to find orphaned/unreferenced objects; deletes them
- GC does not run automatically in the background; must be explicitly scheduled (cron job or manual invocation)
- Stale slices (from compaction) are kept in JuiceFS Trash with a configurable expiration period
- Storage amplification between writes and GC runs: orphaned blocks consume object storage space until GC cleans them
- GC performance: concurrency controlled by `--max-deletes` parameter; tunable
- Relevance for lakehouse: Iceberg-style delete-and-rewrite patterns (e.g., compaction, partition evolution) will accumulate stale blocks; GC schedule must be tuned for these workloads
- Implication: operational discipline required; GC should be scheduled regularly, especially in active lakehouse environments with frequent data rewrites
- Trash feature: deleted files go to Trash for a retention period before permanent deletion; configurable

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Vendor blog; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; appears human-authored
