---
source: https://ozone.apache.org/docs/administrator-guide/operations/s3-multi-tenancy/
component: ozone
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

Apache Ozone supports S3 multi-tenancy, enabling compartmentalized namespaces for multiple tenants in a shared cluster. Each tenant gets an isolated Ozone volume with its own S3 bucket namespace. The feature depends on Apache Ranger for access control enforcement, making Kerberos + Ranger a prerequisite for production multi-tenancy.

## Key Points

- **Before multi-tenancy**: All S3 access via S3G was confined to a single `/s3v` volume — no namespace isolation between teams.
- **With multi-tenancy**: Each tenant gets a unique Ozone volume; buckets can share the same name across tenants because they are in isolated namespaces.
- **Ranger dependency**: Ozone multi-tenancy relies on Apache Ranger to enforce access control. When a tenant is created, Ozone creates Ranger policies on the tenant's volume allowing only bucket owners and tenant admins to access content.
- **S3 credential isolation**: S3 credentials (access key) are tied to a specific Ozone volume; all S3 API requests using that credential are routed and jailed into the corresponding volume.
- **Quota control**: Volumes provide administrative quota allocation at the business-unit level; bucket-level quota for performance and concurrency control.
- **ACL model**: Kerberos principals assigned as tenant users; tenant admins have unrestricted access within their tenancy.
- **Operational complexity**: Requires Kerberos infrastructure + Apache Ranger deployment — significant operational overhead vs. MinIO's simpler access model.

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: none (official Apache documentation)
- Suspicious URLs or redirects: none
- Content quality / AI-generated: official documentation — authoritative
