---
source: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/retries/
component: haproxy
type: article
evidence-tier: vendor
accessible: true
benchmark-age: n/a
date-retrieved: 2026-06-17
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

HAProxy's retries/redispatch tutorial (published by HAProxy Technologies — `vendor` tier). By default
HAProxy retries a failed **connection** 3 times; `retry-on` (2.0+) can extend retries to Layer-7
conditions (timeouts, 5xx, empty/junk responses); `option redispatch` sends the retry to a
**different** server. The docs do **not** address non-idempotent (POST/PUT/DELETE) safety, so an
operator who enables L7 retries can cause a write/delete that already reached a backend to be re-sent
to another backend — dangerous across non-shared MinIO backends.

## Key Points

- **Default**: `retries` = 3, applied to **connection failures**.
- `retry-on` can add: `response-timeout`, `empty-response`, `junk-response`, `conn-failure`,
  `0rtt-rejected`, `all-retryable-errors`, and HTTP 408/425/421/500/502/503/504/404/501.
- `option redispatch` = retry on a **different** server (not the same one) after N failed attempts.
- **Non-idempotent gap**: the documentation does not cover POST/PUT/DELETE safety; retrying a request
  that may already have been processed risks duplicate operations — operators must guard this
  themselves.
- Implication: keep HAProxy retries limited to **pre-send connection failures** (safe), avoid
  `retry-on response-timeout`/5xx + `redispatch` for S3 write/delete traffic, or the LB itself can
  manufacture duplicate/cross-backend mutations.

## Security Notes

No issues detected. Vendor documentation.
Checks performed:
- Malicious or obfuscated code: n/a (documentation).
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; vendor reference, technically neutral.
