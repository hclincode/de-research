---
source: https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/Federation.html
component: hdfs
type: article
evidence-tier: official
accessible: true
benchmark-age: n/a
date-retrieved: 2026-05-21
security:
  malicious-code: none
  suspicious-urls: none
  quality: high
---

## Summary

The official HDFS Federation documentation describes how HDFS Federation adds multiple independent NameNodes/namespaces to a single cluster, allowing horizontal scaling of the metadata tier. ViewFS provides a client-side mount table so applications see a single unified namespace. Router-based Federation (RBF) in Hadoop 3.x improves on basic Federation by adding a routing layer that forwards requests to the correct subcluster, eliminating the operational burden of managing manual namespace partitioning. However, the namespace partition split problem — deciding how to allocate data across NameNodes and managing reallocation — remains a significant operational challenge.

## Key Points

- HDFS Federation allows multiple NameNodes, each responsible for a portion of the namespace, sharing the same DataNode pool.
- ViewFS provides a client-side unified namespace view via configurable mount tables.
- HDFS Router-based Federation (RBF): routers expose a single NameNode interface, route requests to subclusters via a State Store.
- RBF improvements in Hadoop 3.5.0: asynchronous RPC processing, delegation tokens on MySQL (replacing Zookeeper).
- Federation does NOT eliminate NameNode memory limits — it multiplies them by adding NameNodes, each still capped by its RAM.
- The namespace partition problem: which data goes to which NameNode namespace is a manual operational decision; data migration between namespaces requires DistCp-style moves.
- For lakehouse workloads generating billions of small files (Iceberg manifests), Federation only helps if namespace is partitioned by table or workload — requiring careful upfront planning.
- RBF adds operational complexity: State Store (ZooKeeper or MySQL), router nodes, additional failure modes.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: n/a (documentation)
- Suspicious URLs or redirects: none; official hadoop.apache.org domain
- Content quality / AI-generated: high; official Apache project documentation
