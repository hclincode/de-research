---
source: https://github.com/seaweedfs/seaweedfs/wiki/Failover-Master-Server
component: seaweedfs
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

The official Failover Master Server wiki documents how SeaweedFS Raft-based master HA works. The leader manages all volume operations; upon failure, a new leader is elected via Raft and volume servers re-heartbeat to rebuild the leader's view. The documentation acknowledges a temporary writes-unavailability window during transition and does not document split-brain recovery procedures or failover timing guarantees.

## Key Points

- Master coordination: Raft consensus protocol; leader handles all volume management and file ID assignment
- Leader election: automatic upon leader failure; non-leader masters forward requests to leader
- Transition gap: "During the transition, there could be moments where the new leader has partial information about all volume servers" — volumes whose servers have not yet heartbeated to the new leader are temporarily unavailable for writes
- Volume server configuration: best practice is to list all known master servers in `-mserver` flag so volume servers can find any master upon failover
- HA requirement: odd number of masters (1 or 3 typical); production recommendation is 3 for Raft quorum
- Undocumented: split-brain recovery procedures, failover timing guarantees, detailed testing results
- No explicit documentation of how long the partial-view window lasts in practice

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official wiki documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official maintainer documentation
