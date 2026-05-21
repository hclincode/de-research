---
source: https://juicefs.com/docs/community/s3_gateway/
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

Official JuiceFS documentation for deploying the JuiceFS S3 Gateway, which exposes the JuiceFS filesystem as an S3-compatible API endpoint. The gateway is developed based on MinIO Gateway (now deprecated upstream by MinIO) and implements the S3 API. It supports both path-style (default) and virtual-host-style access. Path-style is the default behavior; virtual-host style requires setting the `MINIO_DOMAIN` environment variable.

## Key Points

- JuiceFS S3 Gateway exposes JuiceFS volume as an S3-compatible endpoint
- Developed based on MinIO Gateway code (MinIO deprecated their gateway mode in 2022; JuiceFS maintains a fork)
- Default access style: **path-style** (`http://<host>:<port>/<bucket>/<object>`)
- Virtual-host-style: enabled via `MINIO_DOMAIN` environment variable
- Trino native S3 (`fs.native-s3.enabled=true`) requires setting `s3.path-style-access=true` and `s3.endpoint=http://<juicefs-gateway-host>:<port>`
- Trino officially tests only AWS S3 and MinIO for S3 compatibility — JuiceFS gateway not explicitly listed as tested
- Authentication: S3-style access key / secret key configured at gateway start
- The gateway is a separate process that must be deployed and managed independently of the JuiceFS mount
- Each JuiceFS volume appears as one S3 bucket through the gateway
- Trino legacy Hive connector can also access JuiceFS via the Hadoop Java SDK (jfs:// scheme) without needing the S3 gateway
- S3 gateway adds a hop and potential latency vs. direct Hadoop SDK access

## Security Notes

No issues detected.
Checks performed:
- Malicious or obfuscated code: Official documentation; no code artifacts
- Suspicious URLs or redirects: None
- Content quality / AI-generated: High quality; official project documentation
