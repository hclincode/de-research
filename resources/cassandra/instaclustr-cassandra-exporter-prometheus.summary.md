---
source: https://github.com/instaclustr/cassandra-exporter
component: cassandra
type: github-repo
evidence-tier: vendor-adjacent
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-26
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

Instaclustr's cassandra-exporter is a JVM agent (loaded via JVM_OPTS) that exports Cassandra metrics to Prometheus at a configurable HTTP endpoint (default :9500/metrics). The project documents "Cassandra 4.0+ compatible" with a note that the change broke compatibility with older versions. Instaclustr supports Cassandra 5.0 on their managed platform as of September 2024, and the same agent is used in those environments. Scrape time for all metrics is 10–20 ms in-process, which is significantly faster than remote JMX polling approaches.

## Key Points

- JVM agent approach: loaded with `-javaagent:cassandra-exporter-agent.jar`, no separate process or network hop.
- Cassandra 4.0+ API compatibility; implicitly covers 5.x given Instaclustr's Cassandra 5.0 managed support.
- 10–20 ms scrape time per node; low overhead vs. remote JMX exporter.
- Exposes metrics at `http://localhost:9500/metrics` by default; compatible with standard Prometheus scrape configs.
- Actively maintained by Instaclustr (NetApp subsidiary); last activity more recent than Criteo exporter.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high quality, maintained open-source project
