---
source: https://www.weka.io/resources/technical-brief/why-storage-matters-for-spark-jobs-a-tpc-ds-performance-analysis/
component: weka
type: article
evidence-tier: vendor
accessible: false (gated PDF — content derived from landing page description and secondary references)
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: flagged — vendor-produced marketing material; independent corroboration required
---

## Summary

A Weka-published technical brief claiming that storage is the primary performance bottleneck for Spark jobs, benchmarked using the TPC-DS workload across five Parquet file sizes (128 MB down to 1 MB). The central claim is that NeuralMesh (Weka's filesystem) maintains query performance as file sizes shrink, while alternatives (implied to be S3/NFS) degrade significantly. The actual PDF content is gated behind a lead-generation form and was not accessible for direct analysis — all claims below are derived from page descriptions and secondary references.

## Key Points

- Weka's benchmark targets the "small file problem" — a real and documented Spark/Parquet performance issue where many small files cause excessive metadata operations and reduce scan throughput.
- The brief frames storage as the deciding factor for Spark performance, which is partially true but overstated: compute, memory, shuffle configuration, and query plan optimization are equally or more impactful in most enterprise workloads.
- TPC-DS is a valid analytical benchmark, but it is primarily I/O-read-intensive. It does not stress shuffle (write-heavy) or concurrent multi-tenant workloads, which are common enterprise Spark patterns.
- Weka does not disclose competitor names or configurations in the brief, making the comparison unverifiable.
- No third-party reproductions of this benchmark have been found.

## Security Notes

No issues detected. Content is standard enterprise marketing material.
Checks performed:
- Malicious or obfuscated code: n/a (PDF/HTML marketing page)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: Low signal for objective analysis — vendor-produced, gated, non-reproducible benchmark. Treat all performance claims as directionally indicative, not authoritative.
