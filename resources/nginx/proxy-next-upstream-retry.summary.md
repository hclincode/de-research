---
source: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
component: nginx
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-06-17
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

The nginx `ngx_http_proxy_module` documentation defines `proxy_next_upstream` (used by OpenResty,
which is nginx+Lua). Its **default `error timeout`** means nginx will, on a connection error or
timeout, **retry the request against the next upstream** — and by default this retry **includes the
idempotent methods GET, HEAD, PUT, and DELETE**. Only non-idempotent methods (POST, LOCK, PATCH) are
spared by default (since nginx 1.9.13). This is directly relevant: S3 `PUT` and `DELETE` get retried
across upstreams by default, which across non-shared backends can cause cross-backend writes/deletes.

## Key Points

- `proxy_next_upstream` **default: `error timeout`**. Triggers: `error` (connect/send/read header
  fails), `timeout`, `invalid_header`, optionally `http_5xx`/`http_429` if listed; `http_403`/`http_404`
  are never treated as failures.
- **Idempotent methods (GET, HEAD, PUT, DELETE) are retried by default**; **non-idempotent (POST,
  LOCK, PATCH) are NOT** passed to the next server once sent — unless `non_idempotent` is added.
  → S3 multi-object delete (`POST ?delete`) and multipart-complete (`POST`) are not retried by
  default; **single-object `PUT`/`DELETE` ARE**.
- A request can only move to the next upstream **if nothing has been sent to the client yet**;
  mid-response failures are unrecoverable.
- `proxy_request_buffering` **default `on`** (buffers whole request body before upstream → enables
  retry of large PUTs, adds latency/temp-file use; HTTP/1.1 chunked is always buffered).
  `proxy_buffering` **default `on`**.
- Implication: to avoid cross-backend retry of S3 writes/deletes, set `proxy_next_upstream off` (or
  restrict to pre-send connection errors), and never enable `non_idempotent`.

## Security Notes

No issues detected. Official nginx documentation.
Checks performed:
- Malicious or obfuscated code: n/a (documentation).
- Suspicious URLs or redirects: none.
- Content quality / AI-generated: high; primary vendor/project doc.
