---
source: https://github.com/minio/minio/blob/master/docs/bucket/replication/DESIGN.md
component: minio
type: article
evidence-tier: official
accessible: false
benchmark-age: n/a
date-retrieved: 2026-06-17
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

MinIO bucket/site replication (including active-active) is **asynchronous**: strict consistency
holds **within** a single data center/deployment, but only **eventual consistency** holds **across**
replicated sites. Content here is derived from MinIO replication design/blog material surfaced via
search (not directly fetched — `accessible: false`). The takeaway for this investigation: if a load
balancer spreads reads/writes across two replicated MinIO sites/deployments, read-after-write is
**not** strongly consistent.

## Key Points

- MinIO replication model: "strict consistency within the data center and eventual-consistency across
  the data centers."
- Replication is asynchronous — "asynchronous replication decouples write acknowledgment … background
  worker pools providing eventual consistency"; "near-synchronous" when bandwidth is ample, but still
  not synchronous/linearizable across sites.
- Active-active (two-way) replication keeps sites eventually in sync; automatic failover on GET/HEAD
  fetches a missing object from the peer site — implying a window where an object exists on one site
  but not yet the other.
- Implication: **load-balancing across replicated MinIO sites breaks strong read-after-write
  consistency**; a read routed to the not-yet-replicated peer returns stale/404. Strong consistency
  requires routing a bucket's traffic to a single authoritative deployment.

## Security Notes

No issues detected. Derived from official/vendor MinIO replication material via secondary search
snippets (not directly fetched).
Checks performed:
- Malicious or obfuscated code: n/a.
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; corroborated across multiple MinIO sources.
