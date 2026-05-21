---
source: https://blocksandfiles.com/2025/02/11/weka-specstorage-benchmark/
component: weka
type: article
evidence-tier: press
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — trade press (Blocks & Files); benchmark results published on spec.org
---

## Summary

In January 2025, Weka and HPE achieved No. 1 rankings across all five SPECstorage Solution 2020 benchmark workloads (AI_IMAGE, EDA_BLENDED, GENOMICS, SWBUILD, VDA) using a single consistent configuration on HPE Alletra Storage Server 4110 nodes with Intel Xeon processors. The results were formally submitted to spec.org (publicly verifiable). Key numbers for EDA_BLENDED: 17,000 job sets, 7,650,428 operations/sec, 123,442 MB/s throughput — 2.1x better scalability, 2.45x faster throughput, and 6.5x lower latency than the previous record (NetApp, May 2023). These workloads are HPC-oriented (genomics, EDA, AI image training) — NOT data lakehouse or SQL analytics workloads. SPECstorage does not include a data warehouse or SQL analytics workload.

## Key Points

- **Benchmark authority**: SPECstorage Solution 2020 results are submitted to spec.org and are independently verifiable — this is NOT a vendor self-benchmark. Blocks & Files coverage confirms the submission.
- **Benchmark age**: January 2025 — within the 2-year staleness threshold as of the research date (2026-05-21).
- **Hardware**: HPE Alletra Storage Server 4110 + Intel Xeon. This is a premium NVMe appliance configuration; results do not represent commodity hardware deployments.
- **Workload relevance**: SPECstorage workloads (AI_IMAGE, EDA, GENOMICS, SWBUILD, VDA) simulate HPC and AI storage patterns — many small file reads, streaming sequential I/O, and mixed random/sequential. They do NOT simulate SQL analytics, large Parquet scan workloads, or Iceberg metadata access patterns (LIST-heavy, small object reads).
- **Trino/Spark relevance**: Directionally, Weka's very high IOPS (7.65M ops/sec) and throughput (123 GB/s) suggest strong performance for metadata-heavy lakehouse operations (Iceberg manifest reads, Parquet footer fetches). However, these workloads are not directly comparable — no direct TPC-DS or TPC-H result is publicly available.
- **Key gap**: No SPECstorage SQL analytics workload exists; TPC-DS results remain gated (vendor PDF only). The SPECstorage results strengthen confidence in Weka's raw I/O performance but do not directly validate lakehouse query throughput.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — trade press reporting on publicly verifiable spec.org submission
