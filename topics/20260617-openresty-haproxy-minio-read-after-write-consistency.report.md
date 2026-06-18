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

**Update (field finding): the end-to-end phantom delete persists even with NO replication between the
MinIO clusters.** This rules out replication lag as the sole cause and reframes the root: a phantom
delete is fundamentally a failure of **exactly-once delivery (idempotency)** and **truthful, atomic
acknowledgement** in the proxy chain — both of which break on a *single* cluster with no replication
(at-least-once retries duplicate deletes; lost/premature acks make a *succeeded* commit look failed so
cleanup deletes live files, or make a *lost* write look like a delete). See **Part A′**. Consequence:
removing replication does not remove the phantom delete; the decisive fixes are stopping
cross-backend/non-idempotent retries, eliminating premature/buffered acks, pinning objects to one
consistency domain, and making failures non-destructive (engine settings + bucket versioning).**

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

## Part A′ — Phantom deletes persist even with NO replication: it is an idempotency / ack-atomicity problem, not only a replication-staleness one

**Field finding:** the end-to-end "phantom delete" still occurs **even with no replication between the
MinIO clusters.** That rules out replication lag (A1's second bullet) as the *sole* cause and points
at a deeper layer. The correction to the earlier framing: read-after-write *staleness from
replication* is only **one** way to produce a phantom delete. The phantom delete is more
fundamentally a failure of **(i) exactly-once delivery (idempotency)** and **(ii) atomic, truthful
acknowledgement** across the proxy chain — both of which break on a **single** cluster with no
replication at all.

### Definition and taxonomy of "phantom delete"
A *phantom delete* = an object that the table references is gone (or a DELETE appears at MinIO) without
a single, intended, logical delete of a live file by the engine. Four mechanisms produce it; **none
requires replication**:

| # | Phantom-delete mechanism | Needs replication? | Needs >1 cluster? | Root layer |
|---|---|---|---|---|
| P1 | **Duplicate/replayed DELETE** (at-least-once): one logical delete becomes N physical DELETEs at MinIO | No | No | idempotency |
| P2 | **Live-file deletion via false-FAILURE cleanup**: a *succeeded* commit seen as failed → `cleanAll`/`abort` deletes committed files | No | No | ack-atomicity |
| P3 | **Lost write masquerading as a delete**: a PUT is acked but never durably stored → the object is "missing" exactly as if deleted | No | No | ack-atomicity / durability |
| P4 | **Cross-cluster miss** (independent clusters, non-pinning LB): write lands on cluster A, read/list/delete routed to B that never held it | No (worse without it) | Yes | routing |

### Why each occurs without replication

- **P1 — at-least-once duplication.** Every hop in the chain can turn one request into several:
  OpenResty `proxy_next_upstream` retries idempotent **GET/HEAD/PUT/DELETE** by default
  ([nginx](../resources/nginx/proxy-next-upstream-retry.summary.md)); HAProxy `retry-on`/`redispatch`
  re-sends ([HAProxy](../resources/haproxy/retries-redispatch.summary.md)); the AWS SDK retries
  (Trino `s3.max-error-retries=10`; Iceberg S3 retry); the HMS thrift client retries. On a timeout the
  *response* is lost but the *operation already executed*, so the retry is a second execution. A
  single consistent MinIO makes the duplicate **idempotent on the key** (the object is already gone),
  so P1 alone is usually benign — but it inflates DELETE counts (the "phantom" traffic you see) and
  sets up the timing windows for P2/P3.

- **P2 — false-FAILURE deletes live files (the corruption path, replication-free).** The destructive
  step needs only that a **successful commit be reported as failed**, i.e. an *ambiguous
  acknowledgement*, which the proxy chain manufactures on one cluster: a slow/timed-out
  `CompleteMultipartUpload`, metadata PUT, or HMS `alter_table` whose **ack is lost** even though the
  operation committed. That ambiguity drives `CommitFailedException` (a `CleanableFailure`) →
  `SnapshotProducer.cleanAll()` + `SparkWrite.abort()` delete the now-committed manifests/data (see the
  [corruption report](20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md)).
  *Important refinement for the no-replication case:* the false verdict need not come from a stale
  **S3** read — it can come from **HMS** (a successful `alter_table` whose ack is lost → HMS client
  retry → "table has been modified" → strict status check), or from the proxy returning a non-2xx for
  a request the backend actually applied. The ambiguity lives in the **request path**, not in
  cross-cluster replication.

- **P3 — lost write looks identical to a delete.** If any hop **acknowledges a PUT that did not
  durably land** — OpenResty/HAProxy returning success from a buffer before the backend fsynced, a
  premature `100-continue`/early 200, a retried non-idempotent multipart step that corrupts the
  object, or a write to a backend drive/node that then fails quorum — the engine commits a
  `metadata.json` that references an object **that was never stored**. To a reader this is
  indistinguishable from a deletion: "referenced file not found." No DELETE was ever issued; the
  table is broken anyway. This is purely single-cluster and is *aggravated*, not caused, by the proxy.

- **P4 — independent clusters without object pinning.** "No replication between the MinIO clusters"
  does **not** make the system safer — it makes the clusters fully independent consistency domains. If
  the LB does not deterministically pin each object/bucket to its owning cluster (round-robin /
  least-conn / health-flap reshuffle), a `PUT K` on cluster A is followed by a `GET`/`LIST`/`DELETE` K
  on cluster B that never held K → permanent 404 (replication would at least make it *eventually*
  present). Trino `cleanExtraOutputFiles` listing the data dir on the "wrong" cluster sees a different
  file set and can delete or fail to find committed files; a committed `metadata.json` written to A but
  read via B 404s. Split-brain by routing.

### The underlying invariant
Iceberg's commit + cleanup is safe **iff** the path delivers each mutation **exactly once** and
reports its outcome **truthfully** (success xor failure, never "success reported as failure"). An
unreliable proxy chain provides neither: it is **at-least-once** (retries → P1) and it **obscures the
truth** (lost/premature acks → P2/P3). Those two properties are sufficient to corrupt the table on a
**single cluster with no replication**. Replication staleness (A1) is an *additional*, not the
*necessary*, cause. **Conclusion: removing replication cannot remove the phantom delete; you must fix
idempotency and acknowledgement-atomicity in the path (and, if multiple clusters exist, pin objects to
one).**

### What this changes about the fix
- The replication-specific advice (B2) is **insufficient** by itself.
- The decisive levers become **B3/B4 (stop cross-backend & non-idempotent retries, no premature/buffered
  acks, honest timeouts)** + **B1 (single consistency domain, or P4-style object pinning)** + the
  durability guarantees of MinIO itself (B5), plus the engine-side non-destructive settings and
  **bucket versioning** so P2/P3 are recoverable rather than fatal.

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

**Caveat from the no-replication finding (Part A′):** "strong RAW consistency" is necessary but **not
sufficient** to stop the phantom delete. Even a single perfectly-consistent cluster will corrupt the
table if the path is **not exactly-once and truthfully-acknowledged** — at-least-once retries
duplicate deletes, and lost/premature acks make a succeeded commit look failed (→ live-file deletion)
or a lost write look like a delete. So the requirement list above must add: **(5) no
cross-backend/non-idempotent retries anywhere in the chain (OpenResty, HAProxy, SDK, HMS client), and
(6) no premature/buffered acknowledgement of a write before MinIO has durably stored it.** Strong
consistency + idempotency + honest acks together are what make it possible.

## Evidence Quality

| Claim | Source | Tier | Confidence |
|---|---|---|---|
| MinIO single deployment is strongly RAW/list consistent; requires xfs/zfs/btrfs, not ext4/NFS | [MinIO distributed docs](../resources/minio/distributed-consistency.summary.md) | official | HIGH |
| MinIO cross-site replication is async/eventual | [MinIO replication](../resources/minio/bucket-replication-consistency.summary.md) | official (secondary) | MEDIUM–HIGH |
| nginx retries idempotent PUT/DELETE across upstreams by default (`proxy_next_upstream error timeout`) | [nginx docs](../resources/nginx/proxy-next-upstream-retry.summary.md) | official | HIGH |
| HAProxy redispatch/retry-on can re-send to another server; non-idempotent safety unaddressed | [HAProxy retries](../resources/haproxy/retries-redispatch.summary.md) | vendor | HIGH |
| A stateless LB over non-shared backends cannot be linearizable | distributed-systems theory | analysis | HIGH |
| Versioning converts erroneous deletes into recoverable delete-markers | MinIO/S3 versioning semantics | official (general) | HIGH |
| Phantom delete persists with no replication; root is at-least-once delivery + ambiguous ack (Part A′) | field observation + [corruption report](20260608-iceberg-table-corruption-from-s3-delete-root-cause.report.md) + proxy/SDK retry semantics | analysis + official | HIGH |

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
