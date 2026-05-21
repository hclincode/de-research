---
source: https://www.hpcwire.com/off-the-wire/weka-debuts-neuralmesh-axon-for-exascale-ai-deployments/
component: weka
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high — HPCwire trade press; corroborated by StorageNewsletter, PRNewswire, and BigDATAwire
---

## Summary

Unveiled at RAISE SUMMIT 2025 (July 2025), Weka NeuralMesh Axon is an extension of the NeuralMesh platform that ports Weka's distributed filesystem software to run directly on GPU servers' local NVMe SSDs — creating a "fusion architecture" where GPU servers contribute their local SSDs to a shared storage pool without requiring separate dedicated storage nodes. This is designed for exascale AI deployments and was announced with adoption by Cohere, CoreWeave, and NVIDIA. It claims 20x faster AI performance and 90% GPU utilization. NeuralMesh Axon is primarily an AI-training and inference-focused product; its relevance to analytics/lakehouse workloads is indirect.

## Key Points

- **Architecture shift**: Axon eliminates the need for separate storage servers by using GPU server NVMe SSDs as the storage pool — a "hyperconverged" model for AI infrastructure.
- **Primary target**: Exascale AI model training and inference at neocloud/AI factory scale. Traditional enterprise analytics (Spark, Trino) are a secondary consideration.
- **Availability**: General availability was slated for Fall 2025 as of the July 2025 announcement. As of research date (May 2026), it is expected to be GA but adoption in enterprise analytics environments is likely limited.
- **Lakehouse relevance**: NeuralMesh Axon changes the deployment model — compute and storage are co-located on GPU nodes. For a Spark+Trino lakehouse that uses CPU servers (not GPU servers), NeuralMesh Axon is not the applicable product — the original NeuralMesh on dedicated storage servers is.
- **Adopters named**: Cohere, CoreWeave, NVIDIA — all AI-focused, not traditional lakehouse/analytics users.
- **Product positioning signal**: The direction of Weka's product investment (Axon, exascale AI) suggests that enterprise analytics/lakehouse is not Weka's primary growth target.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high — event-driven press release coverage from HPCwire
