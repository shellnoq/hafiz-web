---
title: Integrations
description: Integrate Hafiz with popular applications and services
---

# Integrations

Hafiz is S3-compatible, making it easy to integrate with hundreds of applications that support Amazon S3 or S3-compatible storage backends.

## Supported Applications

<div class="grid cards" markdown>

-   :material-cloud:{ .lg .middle } __Nextcloud__

    ---

    Self-hosted file sync and share platform.

    [:octicons-arrow-right-24: Nextcloud Guide](nextcloud.md)

-   :material-backup-restore:{ .lg .middle } __Veeam Backup__

    ---

    Enterprise backup and disaster recovery.

    [:octicons-arrow-right-24: Veeam Guide](veeam.md)

-   :material-database:{ .lg .middle } __PostgreSQL / MySQL__

    ---

    Database backup to S3 storage.

    [:octicons-arrow-right-24: Database Backup Guide](database-backup.md)

-   :material-sync:{ .lg .middle } __rclone__

    ---

    Sync files to and from Hafiz.

    [:octicons-arrow-right-24: rclone Guide](rclone.md)

</div>

---

## Quick Integration Guide

Most S3-compatible applications require these settings:

| Setting | Value | Description |
|---------|-------|-------------|
| **Endpoint URL** | `https://your-hafiz-server:9000` | Hafiz S3 API endpoint |
| **Access Key** | Your access key | From admin panel or root credentials |
| **Secret Key** | Your secret key | From admin panel or root credentials |
| **Region** | `us-east-1` | Default region (configurable) |
| **Path Style** | `true` / Enabled | Use path-style URLs (not virtual-hosted) |
| **SSL/TLS** | Depends on setup | Enable for HTTPS connections |

### Path Style vs Virtual Hosted Style

Hafiz uses **path-style** URLs by default:

```
# Path style (Hafiz default)
https://hafiz.example.com:9000/bucket-name/object-key

# Virtual hosted style (AWS default, requires DNS)
https://bucket-name.hafiz.example.com:9000/object-key
```

Most applications support path-style. Look for settings like:
- "Use path-style access"
- "Force path style"
- "Path-based bucket access"

---

## Common Integration Patterns

### Backup and Archive

Use Hafiz as a backup target:

```bash
# Example: Backup with rclone
rclone sync /data/backups hafiz:backup-bucket/daily/

# Example: PostgreSQL backup
pg_dump mydb | gzip | aws --endpoint-url https://hafiz:9000 s3 cp - s3://backups/mydb.sql.gz
```

### Media Storage

Store media files for web applications:

```python
# Django settings.py
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_S3_ENDPOINT_URL = 'https://hafiz.example.com:9000'
AWS_ACCESS_KEY_ID = 'your-access-key'
AWS_SECRET_ACCESS_KEY = 'your-secret-key'
AWS_STORAGE_BUCKET_NAME = 'media'
AWS_S3_ADDRESSING_STYLE = 'path'
```

### Log Aggregation

Send application logs to Hafiz:

```yaml
# Fluentd configuration
<match app.logs.**>
  @type s3
  s3_endpoint https://hafiz.example.com:9000
  s3_bucket app-logs
  aws_key_id your-access-key
  aws_sec_key your-secret-key
  force_path_style true
  path logs/%Y/%m/%d/
</match>
```

### Container Registry Storage

Use Hafiz as Docker registry backend:

```yaml
# Docker Registry config.yml
storage:
  s3:
    accesskey: your-access-key
    secretkey: your-secret-key
    region: us-east-1
    bucket: docker-registry
    endpoint: https://hafiz.example.com:9000
    v4auth: true
    pathstyle: true
```

---

## Compatibility Notes

### Supported S3 Features

| Feature | Status | Notes |
|---------|--------|-------|
| Basic Operations | Full | GET, PUT, DELETE, HEAD, COPY |
| Multipart Upload | Full | For large files >5MB |
| Versioning | Full | Enable per bucket |
| Presigned URLs | Full | For temporary access |
| Server-Side Encryption | Full | SSE-S3 and SSE-C |
| Object Lock (WORM) | Full | Compliance and governance modes |
| Lifecycle Policies | Full | Automatic expiration |
| Bucket Policies | Full | JSON-based access control |
| CORS | Full | Cross-origin requests |
| Tagging | Full | Object metadata |

### Known Limitations

- **Virtual Hosted Style**: Requires DNS wildcard configuration
- **S3 Select**: Not currently supported
- **Glacier Storage Class**: Not applicable (single storage tier)
- **Cross-Region Replication**: Use Hafiz native replication instead

---

## Testing Connectivity

Before configuring an application, test connectivity:

```bash
# Test with AWS CLI
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key

aws --endpoint-url https://hafiz.example.com:9000 s3 ls

# Create test bucket
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://test-bucket

# Upload test file
echo "Hello Hafiz" | aws --endpoint-url https://hafiz.example.com:9000 s3 cp - s3://test-bucket/test.txt

# Download and verify
aws --endpoint-url https://hafiz.example.com:9000 s3 cp s3://test-bucket/test.txt -
```

---

## Need Help?

- Check application-specific guides in this section
- Review [Troubleshooting](../operations/troubleshooting.md) for common issues
- Visit [GitHub Issues](https://github.com/shellnoq/hafiz/issues) for community support
