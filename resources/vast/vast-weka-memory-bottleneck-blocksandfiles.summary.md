---
source: https://www.blocksandfiles.com/ai-ml/2025/02/26/how-weka-and-vast-are-tackling-ai-memory-bottlenecks/1614567
component: vast
type: article
evidence-tier: press
accessible: true
benchmark-age: 2025
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Blocks & Files (February 2025) comparison of VAST and Weka approaches to AI inference memory bottlenecks. Both are narrowly focused on LLM token loading for AI inference — neither discusses analytics, SQL, or lakehouse workloads. WEKA demonstrated 41x prefill time reduction on Llama 3.1 70B; VAST provided no published numbers, only architectural claims. Neither product's lakehouse capabilities are discussed.

## Key Points

- **WEKA approach**: InfiniBand-attached WEKApod with PCIe Gen 5 connects directly to NVIDIA DGX H100; reads aligned to NIC closest to GPU; raw speed focus
- **VAST approach**: VUA (Undivided Attention) agent running on GPU servers; RDMA-attached NVMe; prioritizes token prefix loading by policy for multi-tenant efficiency
- **WEKA benchmark (Llama 3.1 70B)**: Prefill time reduced from 23.97s to 0.58s (41x); data transfer under 0.2s of the 0.58s total — vendor-reported only
- **VAST benchmark**: No published performance numbers in this article; only conceptual claims about multi-tenant prefix sharing
- **Workload scope**: Article is exclusively about AI/LLM inference memory bottlenecks; zero mention of data analytics, SQL, or lakehouse
- **Implication**: Both VAST and Weka are investing R&D in AI inference acceleration, not in analytics/lakehouse tooling — confirms both are AI-first vendors

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: trade press, factual coverage of vendor announcements
