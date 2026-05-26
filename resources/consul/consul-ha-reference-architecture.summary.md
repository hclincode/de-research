---
source: https://developer.hashicorp.com/consul/tutorials/production-vms/reference-architecture
component: consul
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

HashiCorp's official Consul reference architecture recommends 3–5 server nodes for production deployments. The preference between 3 and 5 is driven by fault tolerance requirements: 3 servers tolerate the loss of 1 server (requires 2-node quorum), while 5 servers tolerate the loss of 2 servers simultaneously. For production DCs where a single node failure must not degrade availability, 5 servers across 3 availability zones is the standard recommendation. Consul server nodes should run on dedicated instances because they are more resource-intensive than client agents.

## Key Points

- Minimum for HA: 3 servers (tolerates 1 failure). Recommended for production: 5 servers (tolerates 2 simultaneous failures).
- Raft consensus gets slower with more servers; do not exceed 7 voting servers.
- Consul servers should be on dedicated nodes — they are "more resource intensive than client nodes."
- Consul client agents (lightweight) run on every Cassandra host for local health checks and KV operations.
- For a physical DC without availability zones, distribute 5 servers across distinct physical failure domains (different racks or switches).
- Co-locating Consul servers on Cassandra data nodes is explicitly discouraged by the official architecture guidance.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official HashiCorp documentation
