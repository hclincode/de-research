---
source: https://github.com/seaweedfs/seaweedfs/wiki/Amazon-S3-API
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

The official SeaweedFS Amazon S3 API wiki documents which S3 operations are supported and which are not. Core bucket and object operations are fully implemented, including multipart upload, versioning, lifecycle policies, object locking, and CORS. Several advanced S3 features (S3 Express One Zone, SelectObjectContent, Object Lambda, MFA Delete, lifecycle transition rules) are not implemented.

## Key Points

- Supported bucket ops: CreateBucket, DeleteBucket, HeadBucket, ListBuckets
- Supported object ops: GetObject (with Range and conditional headers), PutObject, CopyObject, DeleteObject, DeleteObjects, HeadObject, ListObjects/V2, ListObjectVersions
- Supported multipart: CreateMultipartUpload, UploadPart, UploadPartCopy, CompleteMultipartUpload, AbortMultipartUpload, ListMultipartUploads, ListParts
- Supported extras: versioning, lifecycle policies (non-transition rules), object locking, CORS, bucket quotas, rate limiting, IAM, OIDC/STS, Server-Side Encryption
- Not supported: ListDirectoryBuckets (S3 Express), GetObjectTorrent, SelectObjectContent, WriteGetObjectResponse, S3 Object Lambda, MFA Delete, lifecycle transition rules, website hosting, replication configuration, analytics/logging features
- Path-style vs virtual-hosted: official wiki does not explicitly document path-style support, but search evidence confirms path-style access (`http://host:8333/bucket/key`) is the primary mode; virtual-hosted-style requires DNS wildcard configuration
- IAM embedded in S3 server since SeaweedFS 3.x; AWS IAM CLI commands work on same endpoint
- OIDC integration: supports Keycloak, Okta, Auth0 with JWT tokens

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: None — official wiki documentation
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High — official maintainer documentation
