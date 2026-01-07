---
title: Nextcloud Integration
description: Configure Nextcloud to use Hafiz as external S3 storage backend
---

# Nextcloud Integration

This guide shows how to integrate Hafiz with [Nextcloud](https://nextcloud.com/) as an external S3-compatible storage backend. You can use Hafiz as:

1. **Primary Storage** - Store all Nextcloud files in Hafiz
2. **External Storage** - Mount Hafiz buckets as folders in Nextcloud

## Table of Contents

- [Prerequisites](#prerequisites)
- [Method 1: External Storage (Recommended)](#method-1-external-storage-recommended)
- [Method 2: Primary Storage](#method-2-primary-storage)
- [Multi-User Setup](#multi-user-setup)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Hafiz Setup

1. **Running Hafiz Server** - Single node or cluster
2. **TLS/HTTPS Enabled** (recommended for production)
3. **Access Credentials** - Access key and secret key
4. **Bucket Created** - Pre-create bucket for Nextcloud

```bash
# Create bucket for Nextcloud
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://nextcloud-storage

# Verify bucket
aws --endpoint-url https://hafiz.example.com:9000 s3 ls
```

### Nextcloud Setup

- Nextcloud 20.0 or later
- PHP with required extensions (`curl`, `xml`, `json`)
- Admin access to Nextcloud

---

## Method 1: External Storage (Recommended)

External storage allows you to mount Hafiz buckets as folders within Nextcloud, while keeping other storage options available.

### Step 1: Enable External Storage App

1. Log in to Nextcloud as administrator
2. Go to **Apps** (top-right menu)
3. Search for **"External storage support"**
4. Click **Enable**

Or via command line:

```bash
sudo -u www-data php /var/www/nextcloud/occ app:enable files_external
```

### Step 2: Configure S3 Storage

1. Go to **Settings** > **Administration** > **External storage**
2. Click **Add storage** and select **Amazon S3**
3. Fill in the configuration:

| Field | Value | Example |
|-------|-------|---------|
| **Folder name** | Display name in Nextcloud | `Hafiz Storage` |
| **Authentication** | Access key | |
| **Configuration** | | |
| - Bucket | Bucket name | `nextcloud-storage` |
| - Hostname** | Hafiz server hostname | `hafiz.example.com` |
| - Port** | Hafiz port | `9000` |
| - Region** | Region code | `us-east-1` |
| - Enable SSL** | Check if using HTTPS | |
| - Enable Path Style** | **Must be checked** | |
| **Access key** | Your Hafiz access key | `hafizadmin` |
| **Secret key** | Your Hafiz secret key | `hafizadmin` |
| **Available for** | Users/groups | All users or specific |

**Important**: Always enable **"Enable Path Style"** as Hafiz uses path-style URLs.

### Step 3: Verify Connection

After saving, Nextcloud shows a colored indicator:
- **Green checkmark** - Connection successful
- **Yellow warning** - Connection issue (hover for details)
- **Red X** - Connection failed

### Step 4: Test File Operations

1. Go to **Files** in Nextcloud
2. Open the **Hafiz Storage** folder (or your chosen name)
3. Upload a test file
4. Verify file appears in Hafiz:

```bash
aws --endpoint-url https://hafiz.example.com:9000 s3 ls s3://nextcloud-storage/
```

---

## Method 2: Primary Storage

Configure Hafiz as Nextcloud's primary storage backend. All files will be stored in Hafiz.

### Step 1: Backup Existing Data

```bash
# Backup Nextcloud data directory
sudo tar -czvf nextcloud-data-backup.tar.gz /var/www/nextcloud/data
```

### Step 2: Edit Nextcloud Configuration

Edit `/var/www/nextcloud/config/config.php`:

```php
<?php
$CONFIG = array (
  // ... existing configuration ...

  'objectstore' => array(
    'class' => '\\OC\\Files\\ObjectStore\\S3',
    'arguments' => array(
      'bucket' => 'nextcloud-primary',
      'hostname' => 'hafiz.example.com',
      'port' => 9000,
      'use_ssl' => true,
      'use_path_style' => true,
      'autocreate' => true,
      'key' => 'your-access-key',
      'secret' => 'your-secret-key',
      'region' => 'us-east-1',
    ),
  ),
);
```

### Step 3: Create Primary Bucket

```bash
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://nextcloud-primary
```

### Step 4: Restart and Test

```bash
# Restart web server
sudo systemctl restart apache2  # or nginx/php-fpm

# Run Nextcloud maintenance
sudo -u www-data php /var/www/nextcloud/occ maintenance:mode --off
sudo -u www-data php /var/www/nextcloud/occ files:scan --all
```

---

## Multi-User Setup

### Create Per-User Buckets

For organizations, create separate buckets per user or department:

```bash
# Create user buckets
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://nc-sales
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://nc-engineering
aws --endpoint-url https://hafiz.example.com:9000 s3 mb s3://nc-hr
```

### Create Hafiz Users

Via Admin Panel or API:

```bash
# Create user with specific bucket access
curl -X POST https://hafiz.example.com:9000/api/v1/users \
  -H "Authorization: Basic $(echo -n 'hafizadmin:hafizadmin' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "sales-dept",
    "description": "Sales Department Storage",
    "bucket_access": [
      {"bucket": "nc-sales", "permission": "readwrite"}
    ]
  }'
```

### Configure External Storage per Group

In Nextcloud External Storage settings:

1. Add storage for `nc-sales` bucket
2. Set **Available for** to `Sales` group
3. Repeat for other departments

---

## Self-Signed Certificates

If using self-signed TLS certificates:

### Option 1: Add CA to Nextcloud Server

```bash
# Copy CA certificate
sudo cp /opt/hafiz/certs/ca.crt /usr/local/share/ca-certificates/hafiz-ca.crt

# Update CA store
sudo update-ca-certificates

# Restart PHP
sudo systemctl restart php8.1-fpm  # or apache2
```

### Option 2: Configure Nextcloud to Accept Self-Signed

Add to `config.php` (not recommended for production):

```php
'objectstore' => array(
  'class' => '\\OC\\Files\\ObjectStore\\S3',
  'arguments' => array(
    // ... other settings ...
    'verify_bucket_exists' => false,
  ),
),
```

---

## Performance Optimization

### Enable Caching

Add to `config.php`:

```php
'memcache.local' => '\\OC\\Memcache\\APCu',
'memcache.distributed' => '\\OC\\Memcache\\Redis',
'redis' => array(
  'host' => 'localhost',
  'port' => 6379,
),
```

### Increase PHP Limits

In `php.ini`:

```ini
memory_limit = 512M
upload_max_filesize = 16G
post_max_size = 16G
max_execution_time = 3600
```

### Configure Multipart Upload

For large files, Nextcloud uses multipart upload. Hafiz handles this automatically.

---

## Example: Complete Setup Script

```bash
#!/bin/bash
# setup-nextcloud-hafiz.sh

HAFIZ_ENDPOINT="https://hafiz.example.com:9000"
HAFIZ_ACCESS_KEY="hafizadmin"
HAFIZ_SECRET_KEY="hafizadmin"
BUCKET_NAME="nextcloud-storage"

# 1. Create bucket
echo "Creating bucket..."
aws --endpoint-url $HAFIZ_ENDPOINT s3 mb s3://$BUCKET_NAME

# 2. Enable versioning (optional but recommended)
echo "Enabling versioning..."
aws --endpoint-url $HAFIZ_ENDPOINT s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# 3. Create test file
echo "Testing connection..."
echo "Hello from Hafiz" | aws --endpoint-url $HAFIZ_ENDPOINT s3 cp - s3://$BUCKET_NAME/test.txt

# 4. Verify
aws --endpoint-url $HAFIZ_ENDPOINT s3 ls s3://$BUCKET_NAME/

echo ""
echo "Bucket ready! Configure Nextcloud with:"
echo "  Hostname: hafiz.example.com"
echo "  Port: 9000"
echo "  Bucket: $BUCKET_NAME"
echo "  Region: us-east-1"
echo "  Access Key: $HAFIZ_ACCESS_KEY"
echo "  Enable SSL: Yes"
echo "  Enable Path Style: Yes"
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection timeout | Network/firewall issue | Check port 9000 is open |
| Access denied | Wrong credentials | Verify access key/secret |
| SSL certificate error | Self-signed cert not trusted | Add CA to server trust store |
| Bucket not found | Bucket doesn't exist | Create bucket first |
| Path style error | Path style not enabled | Enable "Enable Path Style" in Nextcloud |

### Debug Logging

Enable Nextcloud debug logging:

```php
// config.php
'loglevel' => 0,  // Debug level
'log_type' => 'file',
'logfile' => '/var/log/nextcloud.log',
```

Check logs:

```bash
tail -f /var/log/nextcloud.log | grep -i s3
```

### Test with curl

```bash
# Test Hafiz endpoint directly
curl -k https://hafiz.example.com:9000/health

# List buckets with credentials
curl -k "https://hafiz.example.com:9000/" \
  -H "Authorization: AWS4-HMAC-SHA256 ..."  # Use aws cli instead
```

### Verify Bucket Contents

```bash
# Check what Nextcloud stored
aws --endpoint-url https://hafiz.example.com:9000 s3 ls s3://nextcloud-storage/ --recursive

# Download and inspect a file
aws --endpoint-url https://hafiz.example.com:9000 s3 cp s3://nextcloud-storage/files/test.txt -
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Nextcloud + Hafiz Architecture                        │
│                                                                              │
│   ┌──────────────┐                           ┌──────────────────────────────┐│
│   │   Browser    │                           │      Hafiz Cluster           ││
│   │   Client     │                           │                              ││
│   └──────┬───────┘                           │  ┌────────┐    ┌────────┐   ││
│          │                                   │  │ Node 1 │    │ Node 2 │   ││
│          │ HTTPS                             │  └────┬───┘    └────┬───┘   ││
│          ▼                                   │       │             │        ││
│   ┌──────────────┐         S3 API            │       └──────┬──────┘        ││
│   │  Nextcloud   │──────────────────────────▶│              │               ││
│   │   Server     │         HTTPS:9000        │     ┌────────▼────────┐      ││
│   │              │                           │     │    SQLite /     │      ││
│   │ External     │                           │     │   PostgreSQL    │      ││
│   │ Storage App  │                           │     └─────────────────┘      ││
│   └──────────────┘                           │                              ││
│                                              │     ┌─────────────────┐      ││
│   Bucket: nextcloud-storage                  │     │  Object Storage │      ││
│   ├── files/                                 │     │  /opt/hafiz/data│      ││
│   │   ├── user1/                             │     └─────────────────┘      ││
│   │   ├── user2/                             │                              ││
│   │   └── shared/                            └──────────────────────────────┘│
│   └── versions/                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

- [TLS Configuration](../deployment/tls.md) - Secure your Hafiz deployment
- [Cluster Setup](../deployment/cluster.md) - High availability for production
- [Backup Guide](../operations/backup.md) - Backup Nextcloud data stored in Hafiz
