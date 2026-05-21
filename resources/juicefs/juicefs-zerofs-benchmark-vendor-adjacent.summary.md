---
source: https://www.zerofs.net/zerofs-vs-juicefs
component: juicefs
type: article
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: 2024
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: low
---

## Summary

ZeroFS documentation page presenting benchmark comparisons between ZeroFS and JuiceFS. ZeroFS is a competing S3-backed filesystem product; this is a vendor-adjacent source with a direct financial stake in the comparison outcome. The benchmark claims JuiceFS is 175-227x slower than ZeroFS, used 31x more storage, and generated 293x more S3 API calls. The JuiceFS GitHub community disputed the methodology (Discussion #6488), noting the JuiceFS configuration and deployment details were not disclosed, making reproduction impossible. These results cannot be treated as established fact.

## Key Points

- Source: ZeroFS documentation (competing product); vendor-adjacent tier — not independent
- Claimed results: ZeroFS 175-227x faster write throughput than JuiceFS; JuiceFS had 9,206 failures out of 10,000 operations in random-write test
- JuiceFS storage amplification claim: 31x more storage used; 293x more S3 API calls for same workload
- ZeroFS architecture: single-node, writes to SlateDB first then async to S3; not a distributed system — architecturally incomparable to JuiceFS
- JuiceFS community response (Discussion #6488): configuration and metadata engine not disclosed; test may have used default/untuned settings; comparison is not representative of a properly configured JuiceFS deployment
- JuiceFS high failure count in random-write test: may reflect undisclosed network or metadata engine configuration issues, not inherent JuiceFS limitations
- Key methodological gap: which JuiceFS metadata engine was used is unknown; Redis vs TiKV vs PostgreSQL would produce radically different results
- Corroboration needed: no independent source confirms the 175-227x performance gap; treat with LOW confidence
- ZeroFS is a single-node tool vs JuiceFS is a distributed multi-node system — comparing them on raw throughput is architecturally misleading

## Security Notes

Content quality flagged as low due to vendor-competitive framing and undisclosed test methodology.
Checks performed:
- Malicious or obfuscated code: No executable content
- Suspicious URLs or redirects: None
- Content quality / AI-generated: Vendor benchmark; methodology not reproducible
