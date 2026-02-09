---
title: Single Node Deployment
description: Deploy Hafiz on a single server for development, testing, or small-scale production
---

# Single Node Deployment

This guide covers deploying Hafiz on a single server. This is ideal for:

- Development and testing environments
- Small-scale production (< 1TB storage)
- Proof of concept deployments
- Learning and evaluation

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Requirements](#system-requirements)
- [Quick Start](#quick-start)
- [Installation Methods](#installation-methods)
  - [Binary Installation](#binary-installation)
  - [Docker Installation](#docker-installation)
  - [systemd Service](#systemd-service)
- [Configuration](#configuration)
- [TLS/HTTPS Setup](#tlshttps-setup)
- [Verification](#verification)
- [Client Configuration](#client-configuration)
- [Maintenance](#maintenance)
- [Upgrading](#upgrading)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Single Node Hafiz                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Hafiz Server                          ││
│  │                                                          ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ││
│  │  │   S3 API     │  │  Admin API   │  │  Admin Panel │  ││
│  │  │  Port 9000   │  │  /api/v1     │  │    /admin    │  ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘  ││
│  │                                                          ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ││
│  │  │   Storage    │  │   Metadata   │  │     Auth     │  ││
│  │  │  (Local FS)  │  │   (SQLite)   │  │  (Built-in)  │  ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Data Directory: /opt/hafiz/data                            │
│  - objects/        (S3 objects)                             │
│  - hafiz.db        (SQLite metadata)                        │
│  - temp/           (Multipart uploads)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## System Requirements

### Minimum Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 2 cores | 4+ cores |
| RAM | 2 GB | 8+ GB |
| Storage | 20 GB | Based on data needs |
| OS | Linux (kernel 4.x+) | Rocky Linux 9 / Ubuntu 22.04 |

### Supported Operating Systems

- Rocky Linux 8/9 (RHEL compatible)
- Ubuntu 20.04 / 22.04 / 24.04
- Debian 11 / 12
- Amazon Linux 2023
- Any Linux with glibc 2.17+

### Network Requirements

| Port | Protocol | Purpose |
|------|----------|---------|
| 9000 | TCP | S3 API, Admin Panel, Admin API |

---

## Quick Start

### Option 1: One-Line Install (Linux)

```bash
curl -fsSL https://raw.githubusercontent.com/shellnoq/hafiz/main/scripts/install.sh | bash
```

### Option 2: Docker (Fastest)

```bash
docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  ghcr.io/shellnoq/hafiz:latest
```

### Option 3: Manual Binary

```bash
# Download latest release
curl -LO https://github.com/shellnoq/hafiz/releases/latest/download/hafiz-server-linux-amd64
chmod +x hafiz-server-linux-amd64
mv hafiz-server-linux-amd64 /usr/local/bin/hafiz-server

# Create data directory
mkdir -p /opt/hafiz/data

# Run
HAFIZ_DATA_DIR=/opt/hafiz/data hafiz-server
```

---

## Installation Methods

### Binary Installation

#### Step 1: Download Binary

```bash
# Set version
VERSION="0.1.0"

# Download
curl -LO "https://github.com/shellnoq/hafiz/releases/download/v${VERSION}/hafiz-server-linux-amd64"

# Make executable
chmod +x hafiz-server-linux-amd64

# Move to system path
sudo mv hafiz-server-linux-amd64 /usr/local/bin/hafiz-server

# Verify
hafiz-server --version
```

#### Step 2: Create Directories

```bash
# Create base directory
sudo mkdir -p /opt/hafiz/{data,logs,certs}

# Set ownership (replace 'hafiz' with your user)
sudo useradd -r -s /bin/false hafiz
sudo chown -R hafiz:hafiz /opt/hafiz
```

#### Step 3: Create Configuration

```bash
sudo tee /opt/hafiz/data/hafiz.toml << 'EOF'
[server]
bind_address = "0.0.0.0"
port = 9000
admin_port = 9001
workers = 0  # Auto-detect CPU count
max_connections = 10000
request_timeout_secs = 300

[storage]
data_dir = "/opt/hafiz/data"
temp_dir = "/tmp/hafiz"
max_object_size = 5497558138880  # 5 TiB

[database]
url = "sqlite:///opt/hafiz/data/hafiz.db?mode=rwc"
max_connections = 100
min_connections = 5

[auth]
enabled = true
root_access_key = "hafizadmin"
root_secret_key = "hafizadmin"

[logging]
level = "info"
format = "pretty"

[tls]
enabled = false
# Uncomment for HTTPS:
# enabled = true
# cert_file = "/opt/hafiz/certs/server.crt"
# key_file = "/opt/hafiz/certs/server.key"

[cluster]
enabled = false
EOF
```

#### Step 4: Run Manually (Testing)

```bash
HAFIZ_CONFIG_FILE=/opt/hafiz/data/hafiz.toml hafiz-server
```

---

### Docker Installation

#### Basic Docker Run

```bash
docker run -d \
  --name hafiz \
  --restart unless-stopped \
  -p 9000:9000 \
  -v /opt/hafiz/data:/data \
  -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
  -e HAFIZ_ROOT_SECRET_KEY=hafizadmin \
  ghcr.io/shellnoq/hafiz:latest
```

#### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  hafiz:
    image: ghcr.io/shellnoq/hafiz:latest
    container_name: hafiz
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - ./data:/data
      - ./certs:/certs:ro  # For TLS
    environment:
      - HAFIZ_DATA_DIR=/data
      - HAFIZ_DATABASE_URL=sqlite:///data/hafiz.db?mode=rwc
      - HAFIZ_ROOT_ACCESS_KEY=hafizadmin
      - HAFIZ_ROOT_SECRET_KEY=hafizadmin
      - HAFIZ_LOG_LEVEL=info
      # Uncomment for HTTPS:
      # - HAFIZ_TLS_CERT=/certs/server.crt
      # - HAFIZ_TLS_KEY=/certs/server.key
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
  data:
```

```bash
docker compose up -d
```

---

### systemd Service

#### Create Service File

```bash
sudo tee /etc/systemd/system/hafiz.service << 'EOF'
[Unit]
Description=Hafiz S3-Compatible Object Storage Server
Documentation=https://github.com/shellnoq/hafiz
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=hafiz
Group=hafiz
Environment="HAFIZ_CONFIG_FILE=/opt/hafiz/data/hafiz.toml"
ExecStart=/usr/local/bin/hafiz-server
Restart=always
RestartSec=5
LimitNOFILE=65536
WorkingDirectory=/opt/hafiz

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/opt/hafiz/data /opt/hafiz/logs

[Install]
WantedBy=multi-user.target
EOF
```

#### Enable and Start

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable on boot
sudo systemctl enable hafiz

# Start service
sudo systemctl start hafiz

# Check status
sudo systemctl status hafiz

# View logs
sudo journalctl -u hafiz -f
```

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HAFIZ_DATA_DIR` | `/data` | Data storage directory |
| `HAFIZ_DATABASE_URL` | SQLite | Database connection string |
| `HAFIZ_ROOT_ACCESS_KEY` | `hafizadmin` | Root user access key |
| `HAFIZ_ROOT_SECRET_KEY` | `hafizadmin` | Root user secret key |
| `HAFIZ_PORT` | `9000` | S3 API port |
| `HAFIZ_LOG_LEVEL` | `info` | Log level (trace/debug/info/warn/error) |
| `HAFIZ_TLS_CERT` | - | TLS certificate path |
| `HAFIZ_TLS_KEY` | - | TLS private key path |
| `HAFIZ_CONFIG_FILE` | - | Configuration file path |

### Configuration File Options

See [Configuration Guide](../getting-started/configuration.md) for complete reference.

---

## TLS/HTTPS Setup

For secure communication, enable TLS:

### Generate Self-Signed Certificate (Development)

```bash
# Create certificate directory
mkdir -p /opt/hafiz/certs

# Generate private key and certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/hafiz/certs/server.key \
  -out /opt/hafiz/certs/server.crt \
  -subj "/CN=hafiz.local/O=Hafiz" \
  -addext "subjectAltName=DNS:hafiz.local,DNS:localhost,IP:127.0.0.1"

# Set permissions
chmod 600 /opt/hafiz/certs/server.key
chown hafiz:hafiz /opt/hafiz/certs/*
```

### Let's Encrypt Certificate (Production)

```bash
# Install certbot
sudo dnf install -y certbot  # Rocky/RHEL
# or: sudo apt install -y certbot  # Ubuntu/Debian

# Obtain certificate (HTTP challenge)
sudo certbot certonly --standalone \
  -d your-domain.com \
  --agree-tos \
  --email your-email@example.com

# Copy certificates
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem /opt/hafiz/certs/server.crt
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem /opt/hafiz/certs/server.key
sudo chown hafiz:hafiz /opt/hafiz/certs/*

# Set up auto-renewal
echo "0 0 1 * * root certbot renew --quiet && systemctl reload hafiz" | sudo tee /etc/cron.d/hafiz-cert-renewal
```

### Enable TLS

Update configuration:

```bash
# Using environment variables
export HAFIZ_TLS_CERT=/opt/hafiz/certs/server.crt
export HAFIZ_TLS_KEY=/opt/hafiz/certs/server.key

# Or in hafiz.toml
[tls]
enabled = true
cert_file = "/opt/hafiz/certs/server.crt"
key_file = "/opt/hafiz/certs/server.key"
min_version = "1.2"
hsts_enabled = true
```

Restart the service:

```bash
sudo systemctl restart hafiz
```

---

## Verification

### Health Check

```bash
# HTTP
curl http://localhost:9000/health

# HTTPS (with self-signed cert)
curl -k https://localhost:9000/health

# HTTPS (with CA cert)
curl --cacert /opt/hafiz/certs/ca.crt https://localhost:9000/health
```

Expected response: `ok`

### Admin Panel

Open in browser:
- HTTP: `http://your-server:9000/admin`
- HTTPS: `https://your-server:9000/admin`

### S3 API Test

```bash
# Configure credentials
export AWS_ACCESS_KEY_ID=hafizadmin
export AWS_SECRET_ACCESS_KEY=hafizadmin
export AWS_DEFAULT_REGION=us-east-1

# Create bucket
aws --endpoint-url http://localhost:9000 s3 mb s3://test-bucket

# Upload file
echo "Hello Hafiz!" > /tmp/test.txt
aws --endpoint-url http://localhost:9000 s3 cp /tmp/test.txt s3://test-bucket/hello.txt

# Download file
aws --endpoint-url http://localhost:9000 s3 cp s3://test-bucket/hello.txt -

# List objects
aws --endpoint-url http://localhost:9000 s3 ls s3://test-bucket/

# Cleanup
aws --endpoint-url http://localhost:9000 s3 rb s3://test-bucket --force
```

### Full Test Suite (52 tests)

Hafiz includes a comprehensive S3 API test suite that validates all supported operations. The test script is located at `deploy/distributed/tests/run-tests.sh`.

To run against a single-node instance, set the endpoint and credentials:

```bash
export AWS_ACCESS_KEY_ID=hafizadmin
export AWS_SECRET_ACCESS_KEY=hafizadmin
export AWS_DEFAULT_REGION=us-east-1

# Edit endpoint in the script or run with custom endpoint:
EP="--endpoint-url http://localhost:9000"
bash deploy/distributed/tests/run-tests.sh
```

The test suite covers:

| Category | Tests |
|----------|-------|
| Bucket operations | Create, list, head, delete |
| Object operations | Put, get, head, copy, delete, metadata, subfolder |
| Multipart upload | 10MB upload, API-level create/upload/complete/abort |
| Versioning | Enable, put versions, get latest, list versions |
| Bucket/Object tagging | Put, get, delete |
| Lifecycle | Put, get, delete |
| CORS | Put, get, delete |
| SSE-C encryption | Put and get with customer key |

**Verified: 52/52 PASS** on single-node with SQLite backend.

---

## Client Configuration

### AWS CLI

```bash
# Configure profile
aws configure --profile hafiz
# Access Key ID: hafizadmin
# Secret Access Key: hafizadmin
# Default region: us-east-1
# Output format: json

# Usage
aws --endpoint-url http://localhost:9000 --profile hafiz s3 ls
aws --endpoint-url http://localhost:9000 --profile hafiz s3 mb s3://my-bucket
aws --endpoint-url http://localhost:9000 --profile hafiz s3 cp file.txt s3://my-bucket/
```

### Environment Variables

```bash
export AWS_ACCESS_KEY_ID=hafizadmin
export AWS_SECRET_ACCESS_KEY=hafizadmin
export AWS_ENDPOINT_URL=http://localhost:9000

# Now use aws commands without --endpoint-url
aws s3 ls
```

### Python (boto3)

```python
import boto3

s3 = boto3.client(
    's3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='hafizadmin',
    aws_secret_access_key='hafizadmin'
)

# Create bucket
s3.create_bucket(Bucket='my-bucket')

# Upload file
s3.upload_file('local-file.txt', 'my-bucket', 'remote-file.txt')

# List objects
for obj in s3.list_objects_v2(Bucket='my-bucket').get('Contents', []):
    print(obj['Key'])
```

---

## Maintenance

### Backup

```bash
# Stop service (for consistent backup)
sudo systemctl stop hafiz

# Backup data directory
tar -czvf hafiz-backup-$(date +%Y%m%d).tar.gz /opt/hafiz/data

# Restart service
sudo systemctl start hafiz
```

### Restore

```bash
# Stop service
sudo systemctl stop hafiz

# Restore backup
tar -xzvf hafiz-backup-20240115.tar.gz -C /

# Start service
sudo systemctl start hafiz
```

### Log Rotation

```bash
sudo tee /etc/logrotate.d/hafiz << 'EOF'
/opt/hafiz/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    create 0640 hafiz hafiz
    postrotate
        systemctl reload hafiz > /dev/null 2>&1 || true
    endscript
}
EOF
```

---

## Upgrading

### Binary Upgrade

```bash
# Download new version
curl -LO https://github.com/shellnoq/hafiz/releases/latest/download/hafiz-server-linux-amd64

# Stop service
sudo systemctl stop hafiz

# Backup current binary
sudo mv /usr/local/bin/hafiz-server /usr/local/bin/hafiz-server.bak

# Install new binary
sudo mv hafiz-server-linux-amd64 /usr/local/bin/hafiz-server
sudo chmod +x /usr/local/bin/hafiz-server

# Start service
sudo systemctl start hafiz

# Verify
hafiz-server --version
curl http://localhost:9000/health
```

### Docker Upgrade

```bash
# Pull new image
docker pull ghcr.io/shellnoq/hafiz:latest

# Stop and remove old container
docker stop hafiz
docker rm hafiz

# Start with new image
docker run -d \
  --name hafiz \
  --restart unless-stopped \
  -p 9000:9000 \
  -v hafiz-data:/data \
  ghcr.io/shellnoq/hafiz:latest
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Permission denied | Check file ownership: `chown -R hafiz:hafiz /opt/hafiz` |
| Port already in use | Check: `ss -tlnp | grep 9000` |
| Database locked | Ensure only one instance is running |
| TLS handshake failed | Verify certificate paths and permissions |
| Out of disk space | Check: `df -h /opt/hafiz/data` |

### Debug Logging

```bash
# Enable debug logging
HAFIZ_LOG_LEVEL=debug hafiz-server

# Or in systemd
sudo systemctl edit hafiz
# Add: Environment="HAFIZ_LOG_LEVEL=debug"
sudo systemctl restart hafiz
```

### Check Service Status

```bash
# Service status
sudo systemctl status hafiz

# Recent logs
sudo journalctl -u hafiz --since "10 minutes ago"

# Follow logs
sudo journalctl -u hafiz -f
```

### Network Troubleshooting

```bash
# Check if port is listening
ss -tlnp | grep 9000

# Test local connection
curl -v http://localhost:9000/health

# Test from another machine
curl -v http://your-server:9000/health

# Check firewall
sudo firewall-cmd --list-ports  # RHEL/Rocky
sudo ufw status                  # Ubuntu
```

---

## Scaling to Multi-Node Cluster

When you outgrow a single node, Hafiz supports scaling to a multi-node cluster with:

- **Shared PostgreSQL** for consistent metadata across all nodes
- **Native async object replication** for data redundancy (no external tools needed)
- **HAProxy load balancing** with source-based consistent hashing
- **Automatic replication** of all objects to all healthy nodes

See [Multi-Server Cluster Deployment](./cluster.md) for details. The same 52 S3 API tests pass on both single-node and cluster deployments, plus an additional 29 replication tests verify cross-node data consistency.

## Next Steps

- [Configure TLS/HTTPS](./tls.md) for secure communication
- [Set up Multi-Node Cluster](./cluster.md) for high availability
- [Configure External Integrations](../integrations/index.md) (Nextcloud, etc.)
- [Enable Monitoring](../operations/monitoring.md) with Prometheus
