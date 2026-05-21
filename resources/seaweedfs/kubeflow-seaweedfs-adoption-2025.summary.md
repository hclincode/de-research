---
source: https://medium.com/@hpotpose26/kubeflow-pipelines-embraces-seaweedfs-9a7e022d5571
component: seaweedfs
type: article
evidence-tier: press
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Published September 8, 2025, by a Google Summer of Code contributor who implemented the Kubeflow Pipelines SeaweedFS integration (mentored by Julius von Kohout). Kubeflow Pipelines officially adopted SeaweedFS as the default object storage backend, replacing MinIO, to resolve AGPLv3 licensing compliance barriers for CNCF projects. The integration validates SeaweedFS as a drop-in MinIO replacement in a production ML platform context.

## Key Points

- Publication date: September 8, 2025
- Author: Harshvir Potpose (GSoC participant, third-year undergraduate, IT); mentored by Julius von Kohout
- Trigger: MinIO's 2021 AGPLv3 transition prevented KFP from upgrading MinIO versions; SeaweedFS (Apache 2.0) removed this barrier
- Technical implementation: restricted pod security standards (non-root, runAsUser: 1001, dropped Linux capabilities); backward-compatible (`minio-service` retained alongside `seaweedfs` service)
- Multi-tenancy: profile controller dynamically manages namespace-level IAM credentials; per-namespace S3 artifact path isolation
- S3 compatibility: characterized as "drop-in replacement for MinIO" — no breaking changes for existing Kubeflow workloads
- Validation: tested on Kubernetes 1.29.2 and 1.31.0; standalone and multi-user configurations; dedicated namespace-isolation tests
- Deployment status: SeaweedFS becomes default for new Kubeflow releases; MinIO implementation planned for removal
- Limitation: this is Kubeflow's embedded storage use case (ML pipeline artifacts), not a dedicated lakehouse deployment; scale and throughput requirements differ from enterprise lakehouse workloads

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — Medium article
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — detailed technical article with specific version numbers and test configurations
