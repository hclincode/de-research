---
source: https://github.com/seaweedfs/seaweedfs/issues/8151
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

GitHub Issue #8151, opened January 28, 2026, documents a high-severity data corruption bug introduced in SeaweedFS 4.05 or 4.06 when using `weed filer -encryptVolumeData`. The corruption is silent (file sizes appear correct, but content is zeroed or garbage), confirmed by differing MD5 checksums. A linked pull request (#8154) was created but resolution status is not confirmed in the retrieved content.

## Key Points

- Affected versions: 4.05, 4.06, 4.07
- Fixed in: 4.04 (works correctly); unclear if fix was backported or forward-ported in later 4.x releases
- Trigger condition: `weed filer -encryptVolumeData` flag + file uploads through FUSE mount
- Corruption type: silent — file size correct, content truncated/zeroed; MD5 changes (4cb23b27... → 70f868ba...)
- Detection: only discoverable by explicit checksum verification; no in-band error during upload
- Mitigation: avoid `-encryptVolumeData` flag on affected versions; upgrade to version predating or postdating the bug
- Severity: HIGH — silent data corruption in a security-critical feature path
- Note: this bug affects encrypted volume data only; standard (non-encrypted) volume data is not documented as affected by this specific bug

## Security Notes

The bug itself is a data corruption issue, not a security vulnerability. However, it occurs specifically in the encrypted-at-rest feature path, meaning users relying on encryption for compliance may have silently corrupted data.
Checks performed:
- Malicious or obfuscated code: None — GitHub issue page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — tracked GitHub issue with specific reproduction details
