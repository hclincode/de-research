---
source: https://github.com/seaweedfs/seaweedfs/actions/runs/21741699621
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

A SeaweedFS CI workflow run from February 6, 2026 documents a passing Trino Iceberg catalog integration test. The workflow tests "S3 over HTTPS using AWS CLI" — validating SeaweedFS S3 HTTPS compatibility with the AWS CLI toolchain. The test passed in 1 minute 44 seconds total, confirming that basic S3-over-HTTPS compatibility exists in SeaweedFS 4.x as of early 2026.

## Key Points

- Date: February 6, 2026, at 06:55 UTC
- Trigger: pull request synchronization event
- Test scope: Trino Iceberg catalog integration; specifically "test s3 over https using aws-cli"
- Result: PASSED — total runtime 1 minute 44 seconds; `awscli-tests` job ~1 minute 21 seconds
- Protocol tested: HTTPS S3 with AWS CLI
- Significance: confirms that SeaweedFS actively tests Trino Iceberg integration in CI pipeline as of 4.x development cycle
- Limitation: CI tests reflect specific test scenarios, not full production parity; path-style configuration, performance, and edge-case API behavior require separate validation

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — GitHub Actions run page
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — automated CI test result record
