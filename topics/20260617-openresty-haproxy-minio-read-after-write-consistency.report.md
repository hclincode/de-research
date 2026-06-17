---
title: Why OpenResty→HAProxy→MinIO may not guarantee strong read-after-write consistency, and how to fix it
date: 2026-06-17
status: complete
components: [minio, haproxy, openresty, nginx, s3]
constraints:
  - architecture: client → OpenResty (entrypoint) → HAProxy (LB) → MinIO backend(s)
  - goal: understand why strong read-after-write (RAW) consistency can break, and how to guarantee/improve it
extends:
  - 20260617-iceberg-prevent-broken-table-config-inconsistent-store.report.md
  - 20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md
---

## Overview

The Iceberg corruption chain traced earlier depends on the storage path failing to provide strong
read-after-write (RAW) consistency. This report examines the **infrastructure layer** — OpenResty →
HAProxy → MinIO — to answer: *why can this architecture fail to guarantee strong RAW consistency, and
how can it be guaranteed or improved?*

**Verdict: MinIO itself is strongly consistent (read-after-write and list-after-write) within a single
deployment on a proper filesystem — so the storage engine is not inherently the problem. Strong RAW
consistency is lost only through specific, fixable architectural choices: (1) HAProxy fronting
multiple independent or asynchronously-replicated MinIO deployments (a read routed to a different,
not-yet-synced backend than the write returns stale/404); (2) proxy-layer retries/redispatch
re-sending S3 PUT/DELETE across backends (nginx retries idempotent PUT/DELETE by default; HAProxy
redispatch sends to another server); (3) MinIO on a filesystem that voids its guarantee (ext4, NFS);
and (4) timeouts/buffering that turn a successful write into an ambiguous one. It IS possible to
guarantee strong RAW here — by presenting a single strongly-consistent MinIO cluster behind a load
balancer that only spreads connections (no cross-backend retries, no caching) on xfs/zfs/btrfs. The
"cannot guarantee" condition is a property of the current topology, not a law.**

## Background: what strong read-after-write consistency requires

Strong RAW consistency = after a write to object K is acknowledged, every subsequent read of K returns
that write (linearizable per object). This holds only if the read is served from state that reflects
the acknowledged write. A **single MinIO deployment provides this**: "MinIO follows strict
read-after-write and list-after-write consistency model for all i/o operations both in distributed and
standalone modes" ([MinIO distributed docs](../resources/minio/distributed-consistency.summary.md)) —
because every node answers from the same quorum/erasure-coded state. Real AWS S3 has offered the same
since Dec 2020. So consistency exists at the storage layer; the question is whether the **path** in
front of it preserves or breaks it.

## Part A — Why the architecture may NOT guarantee strong RAW consistency

### A1. HAProxy fronting multiple independent / replicated MinIO deployments — the primary break
**Confidence: HIGH.** A stateless load balancer provides linearizability only if all backends share
the *same* authoritative state. Two cases break this:
- **Independent MinIO deployments/pools** (the user described "MinIO **clusters**", plural): each is
  its own consistency domain. A `PUT K` lands on deployment A; a later `GET K` that HAProxy routes to
  deployment B returns 404/stale. No per-deployment guarantee helps across deployments.
- **Replicated sites (active-active)**: MinIO replication is **asynchronous** — "strict consistency
  within the data center and eventual-consistency across the data centers"
  ([MinIO replication](../resources/minio/bucket-replication-consistency.summary.md)). Load-balancing
  reads across replicas yields eventual, not strong, RAW. (Active-active does failover-fetch on
  GET/HEAD for *missing* objects, but that doesn't cover stale overwrites/deletes or LIST.)

### A2. A stateless LB + non-shared state cannot be linearizable
**Confidence: HIGH (theory).** Even with sticky routing, the moment a read can reach a backend that
hasn't applied the write (failover, rebalance, health-flap, or a different hash bucket), the RAW
invariant is violated. Linearizability across independent replicas requires synchronous replication or
single-primary routing — neither is provided by a plain LB over async/independent backends.

### A3. Proxy retries / redispatch re-sending S3 mutations across backends
**Confidence: HIGH.** Both proxies can resend a request to a *different* backend:
- **OpenResty/nginx**: `proxy_next_upstream` defaults to **`error timeout`**, and by default **retries
  idempotent methods GET/HEAD/PUT/DELETE** against the next upstream (only POST/LOCK/PATCH are spared
  unless `non_idempotent` is set) ([nginx docs](../resources/nginx/proxy-next-upstream-retry.summary.md)).
  So an S3 single-object `PUT`/`DELETE` whose first upstream times out is re-sent to another upstream.
  Across independent backends this writes/deletes on the wrong domain or duplicates the mutation;
  combined with a lost ack it is exactly the ambiguity that breaks Iceberg commits.
- **HAProxy**: default `retries 3` on connection failures; `retry-on` can add L7 timeouts/5xx; `option
  redispatch` sends the retry to **another server**. The docs do not address non-idempotent safety
  ([HAProxy retries](../resources/haproxy/retries-redispatch.summary.md)), so enabling L7 retries can
  resend an already-processed `PUT`/`DELETE` to a different backend.

### A4. Timeouts and buffering manufacture ambiguous writes
**Confidence: MEDIUM–HIGH.** `proxy_request_buffering on` (nginx default) buffers the whole body
before contacting MinIO, adding latency for large PUTs/multipart parts; if `timeout server`
(HAProxy) / `proxy_read_timeout` (nginx) is shorter than the real write duration, the client sees a
timeout while the object was actually stored. This is not inconsistency per se, but it produces the
"lost acknowledgement" that the engine then mis-handles (see the corruption report). It also triggers
A3 retries.

### A5. MinIO's own consistency prerequisites are not met
**Confidence: HIGH.** MinIO's guarantee is void on the wrong storage: it requires **xfs/zfs/btrfs**
and explicitly warns **ext4 "does not honor POSIX O_DIRECT/Fdatasync semantics"**, and that **NFS
volumes** underneath a distributed setup are "not guaranteed" to provide the consistency
([MinIO docs](../resources/minio/distributed-consistency.summary.md)). A MinIO cluster on ext4 or NFS
can return stale/RAW-violating reads even with a perfect LB.

### A6. Response caching or body mutation at OpenResty
**Confidence: MEDIUM.** If OpenResty enables `proxy_cache` for S3 responses, GET/HEAD can serve stale
content. If Lua/auth logic rewrites or strips `Content-MD5`/`x-amz-content-sha256`/`Authorization` or
mangles `Expect: 100-continue`, MinIO rejects or mis-handles requests, producing failures that feed
A3/A4. (Caching is usually off for S3, but worth auditing.)

## Part B — How to guarantee or improve strong RAW consistency

### B1. Present a SINGLE strongly-consistent MinIO deployment behind the LB (the real fix)
**Confidence: HIGH.** Make all backends nodes of **one** MinIO distributed cluster (one erasure-coded
deployment), not independent pools. Then HAProxy/OpenResty only spread **connections** across nodes of
the same consistency domain; MinIO guarantees RAW internally, so any node returns the latest write.
This restores strong RAW by construction.

### B2. If multiple sites are required: single-primary routing, not LB-across-replicas
**Confidence: HIGH.** Because cross-site replication is async (A1), route **all** traffic for a bucket
to one **primary** deployment (active-passive). Fail over to the replica only on a real primary
outage, accepting that post-failover you may read slightly stale data until the replica is caught up.
Never load-balance reads across active-active replicas if you need strong RAW.

### B3. HAProxy configuration
**Confidence: HIGH.**
- Keep `retries` limited to **pre-send connection failures**; do **not** add `retry-on response-timeout`
  / 5xx for S3 write/delete traffic; be cautious with `option redispatch` so a half-processed
  `PUT`/`DELETE` is not re-sent to another backend.
- Use `mode http`; set `timeout server`/`timeout client` generously (minutes) to cover large
  PUT/multipart-complete and metadata ops, so successes aren't seen as timeouts.
- If backends must differ, use deterministic routing (`balance source` / stick-tables) as a *partial*
  mitigation — but this is a band-aid; B1 is the real fix.
- Tune health checks (`inter`/`fall`/`rise`) to avoid flapping that reshuffles routing mid-write.

### B4. OpenResty / nginx configuration
**Confidence: HIGH.**
- `proxy_next_upstream off;` (or restrict to connection errors **before** the request is sent) and
  **never** set `non_idempotent` — prevents cross-backend retry of S3 `PUT`/`DELETE`.
- Disable any `proxy_cache` for S3 paths.
- Preserve S3 headers verbatim — do not strip/rewrite `Authorization`, `x-amz-content-sha256`,
  `Content-MD5`, `Host`; use `proxy_http_version 1.1;` and handle `Expect: 100-continue` correctly.
- Set generous `proxy_read_timeout`/`proxy_send_timeout`; size `proxy_buffers` or use
  `proxy_request_buffering off` for very large uploads (note: disabling buffering also removes nginx's
  ability to retry, which is desirable here).

### B5. MinIO storage layer
**Confidence: HIGH.**
- Use **xfs/zfs/btrfs**, never ext4; avoid NFS-backed volumes (A5). Maintain drive/node quorum.
- Enable **bucket versioning** (and optionally object locking/retention) on Iceberg buckets: a delete
  then creates a **delete-marker** and prior versions remain recoverable — turning an erroneous delete
  from "table permanently broken" into "restore the version." This is a powerful safety net that
  complements, not replaces, fixing consistency.

### B6. Verify and defend in depth
**Confidence: HIGH.**
- Add a consistency probe: PUT an object via OpenResty, then immediately GET/HEAD/LIST it many times
  through the full path; assert no 404/stale across repeated reads and (if multi-backend) across
  backends. Run continuously as a canary.
- Combine with the engine-side hardening from the prior report (Spark `s3.delete-enabled=false`, Trino
  `retry-policy=NONE`, HMS `failure.retries=0`) so that, until the path is perfect, failures remain
  non-destructive.

## Is it possible? — direct answer

**Yes — strong read-after-write consistency is achievable for this stack**, because MinIO already
provides it within a single deployment. The requirements are: (1) one MinIO distributed cluster (or
single-primary routing if multi-site), (2) on xfs/zfs/btrfs, not ext4/NFS, behind (3) a load balancer
and entrypoint that only distribute connections and **do not retry/redispatch S3 mutations across
backends** or cache responses, with (4) timeouts long enough to avoid false failures. The current
inability to guarantee consistency is caused by topology/config choices (independent or async-replicated
backends, default proxy retries, possibly ext4/NFS) — all correctable. Until they are corrected,
enable bucket versioning (B5) and the engine-side non-destructive settings (B6) so the residual
inconsistency cannot permanently break the table.

## Evidence Quality

| Claim | Source | Tier | Confidence |
|---|---|---|---|
| MinIO single deployment is strongly RAW/list consistent; requires xfs/zfs/btrfs, not ext4/NFS | [MinIO distributed docs](../resources/minio/distributed-consistency.summary.md) | official | HIGH |
| MinIO cross-site replication is async/eventual | [MinIO replication](../resources/minio/bucket-replication-consistency.summary.md) | official (secondary) | MEDIUM–HIGH |
| nginx retries idempotent PUT/DELETE across upstreams by default (`proxy_next_upstream error timeout`) | [nginx docs](../resources/nginx/proxy-next-upstream-retry.summary.md) | official | HIGH |
| HAProxy redispatch/retry-on can re-send to another server; non-idempotent safety unaddressed | [HAProxy retries](../resources/haproxy/retries-redispatch.summary.md) | vendor | HIGH |
| A stateless LB over non-shared backends cannot be linearizable | distributed-systems theory | analysis | HIGH |
| Versioning converts erroneous deletes into recoverable delete-markers | MinIO/S3 versioning semantics | official (general) | HIGH |

**Gaps / counter-evidence:** MinIO's strong-consistency claim is the vendor's; it is corroborated by
the documented filesystem caveats and a known real-world report that read-after-write "is not
guaranteed" under some conditions (minio/minio issue #20200) — reinforcing that the guarantee is
conditional on correct filesystem + single deployment. The replication-consistency summary was derived
from secondary search snippets (`accessible: false`), not a direct fetch. Whether the user's specific
deployment is multi-cluster, ext4/NFS-backed, or proxy-retry-prone must be confirmed on-site (the
Diagnostics in the corruption report).

## References
- [MinIO distributed consistency](../resources/minio/distributed-consistency.summary.md)
- [MinIO bucket replication consistency](../resources/minio/bucket-replication-consistency.summary.md)
- [nginx proxy_next_upstream retry behavior](../resources/nginx/proxy-next-upstream-retry.summary.md)
- [HAProxy retries & redispatch](../resources/haproxy/retries-redispatch.summary.md)
- Extends: [prevent broken table via config](20260617-iceberg-prevent-broken-table-config-inconsistent-store.report.md),
  [table corruption root cause](20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md)
