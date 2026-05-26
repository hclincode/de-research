---
source: https://www.terraform.io/docs/backends/types/consul.html
component: nomad
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-26
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Terraform supports multiple on-premise state backends: Consul (KV-based, locking via sessions), PostgreSQL (pg backend, locking via advisory locks), and HTTP (custom API). For a stack that already runs Consul for service discovery and Nomad integration, the Consul backend is the zero-additional-infrastructure choice — state is stored at a configurable KV path with automatic locking. The PostgreSQL backend is the correct choice when an existing Postgres instance is available and the team prefers database-native tooling. Local state is not viable for multi-person teams.

## Key Points

- Consul backend: `backend "consul" { address = "consul.example.com:8500"; path = "terraform/state" }`. Locking via Consul session API. No additional infrastructure if Consul is already in the stack.
- PostgreSQL backend: `backend "pg" { conn_str = "postgres://..." }`. Locking via advisory locks. Requires an existing PostgreSQL instance.
- HTTP backend: custom state management endpoint; useful for organizations with existing artifact stores.
- S3-compatible (MinIO on-premise): viable with the S3 backend pointed at a self-hosted MinIO instance — adds infrastructure but is widely understood.
- Recommended choice for this stack: Consul backend (zero additional components). Fallback: PostgreSQL backend if a Postgres instance already exists for Reaper's storage.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official HashiCorp documentation
