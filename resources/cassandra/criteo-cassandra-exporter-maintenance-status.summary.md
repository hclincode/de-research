---
source: https://github.com/criteo/cassandra_exporter
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

Criteo's cassandra_exporter is a standalone Prometheus exporter for Apache Cassandra forked from the Prometheus JMX exporter. The latest release is 2.3.8, dated March 26, 2022 — over four years old as of this research date. Documented version support lists 2.2.x, 3.x, 4.x with no explicit 5.x mention. The project shows signs of abandonment: 12 open issues, 3 open PRs, no release since 2022. Not recommended as the primary exporter for Cassandra 5.x.

## Key Points

- Last release March 2022; four-year gap is a significant abandonment signal.
- Supports Cassandra 2.2.x / 3.x / 4.x; Cassandra 5.x compatibility unconfirmed.
- Standalone Java process, not a JVM agent — scrapes JMX remotely, which adds network hop overhead vs. in-process agents.
- Not recommended for new Cassandra 5.x deployments; use prometheus/jmx_exporter or instaclustr/cassandra-exporter instead.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high quality, open-source repository
