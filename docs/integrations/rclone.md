---
title: rclone Integration
description: Sync files to and from Hafiz using rclone
---

# rclone Integration

[rclone](https://rclone.org/) is a command-line program to sync files and directories to and from various cloud storage providers, including S3-compatible storage like Hafiz.

## Quick Setup

### Install rclone

```bash
# Linux
curl https://rclone.org/install.sh | sudo bash

# macOS
brew install rclone

# Windows
choco install rclone
```

### Configure Hafiz Remote

```bash
rclone config
```

Choose the following options:
1. `n` - New remote
2. Name: `hafiz`
3. Storage type: `s3`
4. Provider: `Other`
5. Access Key ID: `your-access-key`
6. Secret Access Key: `your-secret-key`
7. Region: `us-east-1`
8. Endpoint: `https://hafiz.example.com:9000`
9. Accept defaults for other options

### Alternative: Config File

Add to `~/.config/rclone/rclone.conf`:

```ini
[hafiz]
type = s3
provider = Other
access_key_id = your-access-key
secret_access_key = your-secret-key
endpoint = https://hafiz.example.com:9000
```

## Common Commands

### List Buckets

```bash
rclone lsd hafiz:
```

### List Files

```bash
rclone ls hafiz:my-bucket
```

### Sync Directory to Hafiz

```bash
# One-way sync (source to destination)
rclone sync /local/path hafiz:bucket-name/path

# With progress
rclone sync /local/path hafiz:bucket-name --progress

# Dry run first
rclone sync /local/path hafiz:bucket-name --dry-run
```

### Copy Files

```bash
# Copy single file
rclone copy /local/file.txt hafiz:bucket-name/

# Copy directory
rclone copy /local/dir hafiz:bucket-name/dir
```

### Download from Hafiz

```bash
rclone copy hafiz:bucket-name/file.txt /local/path/
```

### Bidirectional Sync

```bash
rclone bisync /local/path hafiz:bucket-name
```

## Backup Script Example

```bash
#!/bin/bash
# backup-to-hafiz.sh

BACKUP_DIR="/data/backups"
BUCKET="backups"
DATE=$(date +%Y-%m-%d)

# Sync with checksum verification
rclone sync $BACKUP_DIR hafiz:$BUCKET/$DATE \
  --checksum \
  --transfers 4 \
  --checkers 8 \
  --log-file /var/log/rclone-backup.log \
  --log-level INFO

echo "Backup completed: $DATE"
```

## Self-Signed Certificates

For self-signed TLS certificates:

```ini
[hafiz]
type = s3
provider = Other
access_key_id = your-access-key
secret_access_key = your-secret-key
endpoint = https://hafiz.example.com:9000
no_check_certificate = true
```

Or use the CA certificate:

```bash
export SSL_CERT_FILE=/opt/hafiz/certs/ca.crt
rclone ls hafiz:
```

## Performance Tuning

```bash
rclone sync /source hafiz:bucket \
  --transfers 16 \       # Parallel transfers
  --checkers 32 \        # Parallel checks
  --s3-chunk-size 64M \  # Multipart chunk size
  --s3-upload-concurrency 8
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Certificate error | Add `no_check_certificate = true` or set `SSL_CERT_FILE` |
| Access denied | Verify access key and secret key |
| Bucket not found | Create bucket first or check name |
| Slow transfers | Increase `--transfers` and `--checkers` |

---

For more details, see the [rclone S3 documentation](https://rclone.org/s3/).
