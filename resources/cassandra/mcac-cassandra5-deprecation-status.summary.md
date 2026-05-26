---
source: https://github.com/datastax/metric-collector-for-apache-cassandra/releases
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

MCAC (Metric Collector for Apache Cassandra) latest release is v0.3.6 (March 31, 2025). The documented supported versions are Cassandra 2.2.x through 4.1; Cassandra 5.x is not explicitly listed. The project is in maintenance mode: v0.3.3 added "Cassandra 4.1 and newer" support but there is no explicit 5.x announcement. The K8ssandra project has signaled MCAC will be deprecated in a future release in favor of the Management API for Apache Cassandra (MAAC) and a new Vector-based metrics endpoint.

## Key Points

- Latest release v0.3.6, March 2025 — project is alive but infrequently updated (42 open issues, 12 PRs).
- Explicit Cassandra 5.x support is undocumented; "4.1 and newer" language in v0.3.3 may cover 5.x but is unconfirmed.
- K8ssandra ecosystem is replacing MCAC with MAAC + Vector integration; MCAC will be removed in a future K8ssandra release.
- For non-K8s deployments, MCAC's JVM agent approach still functions if the Dropwizard metrics API has not broken compatibility in Cassandra 5.x — but this is not confirmed in release notes.
- The safer alternative for Cassandra 5.x on bare metal is the prometheus/jmx_exporter (actively maintained by Prometheus project) with Cassandra-specific YAML rules, or instaclustr/cassandra-exporter (last release 2022, Cassandra 4.0+ API).

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none
- Suspicious URLs or redirects: none
- Content quality / AI-generated: high quality, official release history
