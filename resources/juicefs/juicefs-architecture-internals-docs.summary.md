---
source: https://juicefs.com/docs/community/internals/
component: juicefs
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

Official JuiceFS Community Edition documentation describing internal architecture. JuiceFS decouples file data from metadata: data is stored as blocks in object storage, and metadata (file names, sizes, ownership, chunk-to-block mappings) is stored in an independent metadata engine (database). The architecture consists of three components: JuiceFS client, metadata engine, and data storage. Metadata engine and object storage are completely decoupled from each other.

## Key Points

- Three-component architecture: (1) JuiceFS client, (2) metadata engine, (3) object storage backend
- File data is decomposed into chunks (max 64 MiB), slices, and blocks (max 4 MiB); blocks are uploaded as independent objects
- All block-to-file mappings, file names, sizes, permissions, and inode info stored in metadata engine
- Metadata engine options: Redis (in-memory), SQL (MySQL, PostgreSQL, SQLite), TKV (TiKV, BadgerDB)
- JuiceFS client handles all protocol translation; no server-side compute component for data path
- Data flows directly from client to object storage; metadata engine only handles filesystem semantics
- Close-to-open consistency: changes visible to other clients after the writing client closes the file
- Write flow: data written to local write buffer → uploaded to object storage → metadata updated atomically
- Read flow: block fetched from object storage (or local cache hit); metadata resolved first via metadata engine
- Local cache (read cache + write buffer) is client-side; Enterprise Edition adds distributed cache

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation
