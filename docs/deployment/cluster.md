---
title: Multi-Server Cluster Deployment
description: Deploy Hafiz across multiple physical servers
---

# Multi-Server Cluster Deployment

This guide covers deploying Hafiz across multiple physical servers with shared PostgreSQL metadata.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Single-Network Cluster](#step-1-postgresql-setup)
- [Adding Servers to Existing Cluster](#adding-servers-to-existing-cluster)
- [Cross-Network Replication](#cross-network-replication)
- [Air-Gapped System Replication](#air-gapped-system-replication)

## Architecture Overview

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │  (HAProxy/Nginx)│
                    │  dev-hafiz.e2e.lab
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   Server 1    │   │   Server 2    │   │   Server 3    │
│  Hafiz Node   │   │  Hafiz Node   │   │  Hafiz Node   │
│  192.168.1.10 │   │  192.168.1.11 │   │  192.168.1.12 │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                    ┌───────▼───────┐
                    │  PostgreSQL   │
                    │  192.168.1.5  │
                    └───────────────┘
```

## Prerequisites

- 3+ servers with Rocky Linux 8/9 or RHEL 8/9
- PostgreSQL 13+ on a dedicated server (or managed service)
- Network connectivity between all nodes
- Domain name with DNS configured (e.g., `dev-hafiz.e2e.lab`)

## Step 1: PostgreSQL Setup

On the PostgreSQL server (192.168.1.5):

```bash
# Install PostgreSQL
sudo dnf install -y postgresql16-server postgresql16

# Initialize and start
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql

# Create database and user
sudo -u postgres psql << 'EOF'
CREATE USER hafiz WITH PASSWORD 'your_strong_password';
CREATE DATABASE hafiz OWNER hafiz;
GRANT ALL PRIVILEGES ON DATABASE hafiz TO hafiz;
EOF

# Allow remote connections - edit pg_hba.conf
echo "host hafiz hafiz 192.168.1.0/24 md5" | sudo tee -a /var/lib/pgsql/16/data/pg_hba.conf

# Edit postgresql.conf
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /var/lib/pgsql/16/data/postgresql.conf

# Restart PostgreSQL
sudo systemctl restart postgresql

# Open firewall
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

## Step 2: Deploy Hafiz Nodes

On each Hafiz server (192.168.1.10, .11, .12):

```bash
# Download deployment script
curl -O https://raw.githubusercontent.com/shellnoq/hafiz/main/deploy/rocky/deploy.sh
chmod +x deploy.sh

# Set environment variables
export HAFIZ_DATABASE_URL="postgresql://hafiz:your_strong_password@192.168.1.5:5432/hafiz"
export HAFIZ_ROOT_ACCESS_KEY="hafizadmin"
export HAFIZ_ROOT_SECRET_KEY="your_secret_key"
export HAFIZ_S3_BIND="0.0.0.0"
export HAFIZ_S3_PORT="9000"

# Run single node (not cluster compose)
sudo ./deploy.sh single
```

Or with Docker directly:

```bash
docker run -d \
  --name hafiz \
  --restart unless-stopped \
  -p 9000:9000 \
  -v /data/hafiz:/data \
  -e HAFIZ_DATABASE_URL="postgresql://hafiz:your_password@192.168.1.5:5432/hafiz" \
  -e HAFIZ_ROOT_ACCESS_KEY="hafizadmin" \
  -e HAFIZ_ROOT_SECRET_KEY="your_secret_key" \
  hafiz:latest
```

## Step 3: Load Balancer Setup

### Option A: HAProxy (Recommended)

On the load balancer server:

```bash
sudo dnf install -y haproxy
```

Edit `/etc/haproxy/haproxy.cfg`:

```haproxy
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend hafiz_frontend
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/hafiz.pem
    redirect scheme https code 301 if !{ ssl_fc }
    default_backend hafiz_backend

backend hafiz_backend
    balance roundrobin
    option httpchk GET /metrics
    http-check expect status 200

    server node1 192.168.1.10:9000 check
    server node2 192.168.1.11:9000 check
    server node3 192.168.1.12:9000 check

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats auth admin:your_stats_password
```

```bash
sudo systemctl enable --now haproxy
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload
```

### Option B: Nginx

```nginx
upstream hafiz_cluster {
    least_conn;
    server 192.168.1.10:9000 weight=1;
    server 192.168.1.11:9000 weight=1;
    server 192.168.1.12:9000 weight=1;
}

server {
    listen 80;
    server_name dev-hafiz.e2e.lab;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name dev-hafiz.e2e.lab;

    ssl_certificate /etc/nginx/ssl/hafiz.crt;
    ssl_certificate_key /etc/nginx/ssl/hafiz.key;

    client_max_body_size 5G;

    location / {
        proxy_pass http://hafiz_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # For large file uploads
        proxy_request_buffering off;
        proxy_buffering off;
    }
}
```

## Step 4: SSL Certificate

### Let's Encrypt (Production)

```bash
sudo dnf install -y certbot
sudo certbot certonly --standalone -d dev-hafiz.e2e.lab

# For HAProxy, combine cert and key
sudo cat /etc/letsencrypt/live/dev-hafiz.e2e.lab/fullchain.pem \
        /etc/letsencrypt/live/dev-hafiz.e2e.lab/privkey.pem \
    > /etc/haproxy/certs/hafiz.pem
```

### Self-Signed (Development)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/haproxy/certs/hafiz.key \
    -out /etc/haproxy/certs/hafiz.crt \
    -subj "/CN=dev-hafiz.e2e.lab"

cat /etc/haproxy/certs/hafiz.crt /etc/haproxy/certs/hafiz.key > /etc/haproxy/certs/hafiz.pem
```

## Step 5: DNS Configuration

Add DNS records:

```
dev-hafiz.e2e.lab    A    192.168.1.100  # Load balancer IP
```

Or add to `/etc/hosts` on client machines:

```
192.168.1.100  dev-hafiz.e2e.lab
```

## Verification

```bash
# Test S3 API
aws --endpoint-url https://dev-hafiz.e2e.lab s3 ls

# Create bucket
aws --endpoint-url https://dev-hafiz.e2e.lab s3 mb s3://mybucket

# Upload file
aws --endpoint-url https://dev-hafiz.e2e.lab s3 cp test.txt s3://mybucket/

# Access Admin UI
open https://dev-hafiz.e2e.lab/admin
```

## Monitoring

All nodes export Prometheus metrics at `/metrics`:

```bash
curl https://dev-hafiz.e2e.lab/metrics
```

Add to Prometheus:

```yaml
scrape_configs:
  - job_name: 'hafiz'
    static_configs:
      - targets:
        - '192.168.1.10:9000'
        - '192.168.1.11:9000'
        - '192.168.1.12:9000'
```

## Troubleshooting

### Nodes not syncing

All nodes must connect to the same PostgreSQL database. Verify:

```bash
# On each node
docker logs hafiz 2>&1 | grep -i postgres
# Should show: "Using PostgreSQL backend"
```

### Connection refused

Check firewall rules:

```bash
sudo firewall-cmd --list-all
# Ensure port 9000 is open
```

### Load balancer health checks failing

```bash
# Test from load balancer
curl http://192.168.1.10:9000/metrics
```

---

## Adding Servers to Existing Cluster

When scaling your Hafiz cluster, you can add new servers at any time. All nodes share the same PostgreSQL database, so new nodes automatically have access to all metadata.

### Step 1: Prepare the New Server

On the new server (e.g., 192.168.1.13):

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker

# Open firewall ports
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --reload
```

### Step 2: Test PostgreSQL Connectivity

```bash
# Test connection to shared PostgreSQL
psql postgresql://hafiz:your_password@192.168.1.5:5432/hafiz -c "SELECT 1"
```

### Step 3: Deploy Hafiz Node

```bash
# Deploy with the same configuration as existing nodes
docker run -d \
  --name hafiz \
  --restart unless-stopped \
  -p 9000:9000 \
  -v /data/hafiz:/data \
  -e HAFIZ_DATABASE_URL="postgresql://hafiz:your_password@192.168.1.5:5432/hafiz" \
  -e HAFIZ_ROOT_ACCESS_KEY="hafizadmin" \
  -e HAFIZ_ROOT_SECRET_KEY="your_secret_key" \
  -e HAFIZ_STORAGE_BASE_PATH="/data/objects" \
  hafiz:latest
```

### Step 4: Update Load Balancer

Add the new server to HAProxy:

```haproxy
backend hafiz_backend
    balance roundrobin
    option httpchk GET /metrics
    http-check expect status 200

    server node1 192.168.1.10:9000 check
    server node2 192.168.1.11:9000 check
    server node3 192.168.1.12:9000 check
    server node4 192.168.1.13:9000 check  # New node
```

Reload HAProxy:

```bash
sudo systemctl reload haproxy
```

### Step 5: Verify the New Node

```bash
# Check node health
curl http://192.168.1.13:9000/metrics

# Verify in HAProxy stats
curl http://localhost:8404/stats
```

### Adding Multiple Nodes Simultaneously

You can add multiple servers at once using a deployment script:

```bash
#!/bin/bash
# deploy-nodes.sh

NODES=("192.168.1.14" "192.168.1.15" "192.168.1.16")
POSTGRES_URL="postgresql://hafiz:your_password@192.168.1.5:5432/hafiz"
ACCESS_KEY="hafizadmin"
SECRET_KEY="your_secret_key"

for node in "${NODES[@]}"; do
    echo "Deploying to $node..."
    ssh root@$node "docker run -d \
        --name hafiz \
        --restart unless-stopped \
        -p 9000:9000 \
        -v /data/hafiz:/data \
        -e HAFIZ_DATABASE_URL='$POSTGRES_URL' \
        -e HAFIZ_ROOT_ACCESS_KEY='$ACCESS_KEY' \
        -e HAFIZ_ROOT_SECRET_KEY='$SECRET_KEY' \
        -e HAFIZ_STORAGE_BASE_PATH='/data/objects' \
        hafiz:latest"
done

echo "All nodes deployed!"
```

---

## Cross-Network Replication

Connect Hafiz clusters across different networks (data centers, cloud regions, or office locations) for disaster recovery and geographic distribution.

### Architecture: Cross-Network Setup

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Network A (Primary)                         │
│                         (192.168.1.0/24)                            │
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │   Node A1   │     │   Node A2   │     │ PostgreSQL  │          │
│   │ 192.168.1.10│     │ 192.168.1.11│     │ 192.168.1.5 │          │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘          │
│          │                   │                   │                   │
│          └───────────────────┴───────────────────┘                   │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │    VPN / WireGuard   │
                    │   or Site-to-Site    │
                    └──────────┬──────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                              │                                       │
│   ┌──────────────────────────┴──────────────────────────┐           │
│   │                                                      │           │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │   Node B1   │     │   Node B2   │     │PostgreSQL   │          │
│   │ 10.0.1.10   │     │ 10.0.1.11   │     │ (Replica)   │          │
│   └─────────────┘     └─────────────┘     │ 10.0.1.5    │          │
│                                           └─────────────┘           │
│                                                                      │
│                         Network B (Secondary)                        │
│                           (10.0.1.0/24)                             │
└─────────────────────────────────────────────────────────────────────┘
```

### Option 1: PostgreSQL Logical Replication

Use PostgreSQL's built-in logical replication for real-time metadata sync.

#### Primary Site (Network A)

```bash
# Edit postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4

# Create replication user
sudo -u postgres psql << 'EOF'
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'repl_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
CREATE PUBLICATION hafiz_pub FOR ALL TABLES;
EOF

# Update pg_hba.conf for remote access
echo "host hafiz replication_user 10.0.1.0/24 md5" >> pg_hba.conf

# Restart PostgreSQL
sudo systemctl restart postgresql
```

#### Secondary Site (Network B)

```bash
# Create database (without initial data)
sudo -u postgres psql << 'EOF'
CREATE DATABASE hafiz;
CREATE SUBSCRIPTION hafiz_sub
    CONNECTION 'host=192.168.1.5 port=5432 dbname=hafiz user=replication_user password=repl_password'
    PUBLICATION hafiz_pub;
EOF
```

#### Deploy Secondary Nodes

```bash
# On secondary nodes, connect to local PostgreSQL replica
docker run -d \
  --name hafiz \
  --restart unless-stopped \
  -p 9000:9000 \
  -v /data/hafiz:/data \
  -e HAFIZ_DATABASE_URL="postgresql://hafiz:password@10.0.1.5:5432/hafiz" \
  -e HAFIZ_ROOT_ACCESS_KEY="hafizadmin" \
  -e HAFIZ_ROOT_SECRET_KEY="your_secret_key" \
  hafiz:latest
```

### Option 2: Active-Active with HAProxy

For active-active setup with both sites serving traffic:

#### Global HAProxy Configuration

```haproxy
global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5000
    timeout client 30000
    timeout server 30000

# Primary site backend
backend site_a
    balance roundrobin
    option httpchk GET /metrics
    server a1 192.168.1.10:9000 check
    server a2 192.168.1.11:9000 check

# Secondary site backend
backend site_b
    balance roundrobin
    option httpchk GET /metrics
    server b1 10.0.1.10:9000 check
    server b2 10.0.1.11:9000 check

# Active-active frontend with failover
frontend hafiz_global
    bind *:9000

    # Health-based routing
    acl site_a_up nbsrv(site_a) gt 0
    acl site_b_up nbsrv(site_b) gt 0

    # Prefer primary site
    use_backend site_a if site_a_up
    use_backend site_b if site_b_up !site_a_up

    default_backend site_a
```

### Option 3: S3 Bucket Replication

Replicate object data between clusters using S3-compatible replication:

```bash
#!/bin/bash
# sync-clusters.sh - Run via cron every 5 minutes

SOURCE_ENDPOINT="https://hafiz-primary.example.com"
TARGET_ENDPOINT="https://hafiz-secondary.example.com"
BUCKETS=("important-data" "backups" "archives")

for bucket in "${BUCKETS[@]}"; do
    echo "Syncing $bucket..."
    aws --endpoint-url $SOURCE_ENDPOINT s3 sync \
        s3://$bucket /tmp/sync-$bucket/ \
        --delete

    aws --endpoint-url $TARGET_ENDPOINT s3 sync \
        /tmp/sync-$bucket/ s3://$bucket \
        --delete

    rm -rf /tmp/sync-$bucket/
done
```

For real-time replication, use `rclone` with checksums:

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure endpoints
rclone config create primary s3 \
    provider=Other \
    endpoint=https://hafiz-primary.example.com \
    access_key_id=hafizadmin \
    secret_access_key=your_secret

rclone config create secondary s3 \
    provider=Other \
    endpoint=https://hafiz-secondary.example.com \
    access_key_id=hafizadmin \
    secret_access_key=your_secret

# Real-time sync with inotify
rclone sync primary:mybucket secondary:mybucket --checksum --progress
```

### Network Requirements

| Protocol | Port | Direction | Purpose |
|----------|------|-----------|---------|
| TCP | 5432 | Bidirectional | PostgreSQL replication |
| TCP | 9000 | Bidirectional | S3 API traffic |
| UDP | 51820 | Bidirectional | WireGuard VPN |
| TCP | 22 | Bidirectional | SSH management |

### WireGuard VPN Setup

For secure cross-network connectivity:

#### Site A (Primary)

```bash
# Install WireGuard
sudo dnf install -y wireguard-tools

# Generate keys
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key

# Configure /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <site_a_private_key>
Address = 10.200.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <site_b_public_key>
AllowedIPs = 10.200.0.2/32, 10.0.1.0/24
Endpoint = site-b-public-ip:51820
PersistentKeepalive = 25
```

#### Site B (Secondary)

```bash
# Configure /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <site_b_private_key>
Address = 10.200.0.2/24
ListenPort = 51820

[Peer]
PublicKey = <site_a_public_key>
AllowedIPs = 10.200.0.1/32, 192.168.1.0/24
Endpoint = site-a-public-ip:51820
PersistentKeepalive = 25
```

```bash
# Enable on both sites
sudo systemctl enable --now wg-quick@wg0
```

---

## Air-Gapped System Replication

For environments with no network connectivity (classified networks, secure facilities, disaster recovery sites), Hafiz supports offline data transfer.

### Architecture: Air-Gapped Setup

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Secure Network (Source)                          │
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │  Hafiz A1   │     │  Hafiz A2   │     │ PostgreSQL  │          │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘          │
│          └───────────────────┴───────────────────┘                   │
│                              │                                       │
│                    ┌─────────▼─────────┐                            │
│                    │   Export Server   │                            │
│                    │ (Data Diode Out)  │                            │
│                    └─────────┬─────────┘                            │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Physical Media    │
                    │  (USB/Tape/Drive)   │
                    └──────────┬──────────┘
                               │
┌──────────────────────────────┼───────────────────────────────────────┐
│                    ┌─────────▼─────────┐                            │
│                    │  Import Server    │                            │
│                    │ (Data Diode In)   │                            │
│                    └─────────┬─────────┘                            │
│                              │                                       │
│          ┌───────────────────┴───────────────────┐                   │
│          │                   │                   │                   │
│   ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐          │
│   │  Hafiz B1   │     │  Hafiz B2   │     │ PostgreSQL  │          │
│   └─────────────┘     └─────────────┘     └─────────────┘           │
│                                                                      │
│                    Air-Gapped Network (Target)                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Export Data from Source Cluster

#### 1. Create Export Script

```bash
#!/bin/bash
# hafiz-export.sh - Export Hafiz data for air-gapped transfer

set -e

EXPORT_DIR="/mnt/export/hafiz-$(date +%Y%m%d-%H%M%S)"
POSTGRES_URL="postgresql://hafiz:password@localhost:5432/hafiz"
S3_ENDPOINT="http://localhost:9000"

mkdir -p "$EXPORT_DIR"/{metadata,objects,checksums}

echo "=== Hafiz Air-Gapped Export ==="
echo "Export directory: $EXPORT_DIR"
echo "Timestamp: $(date -Iseconds)"

# 1. Export PostgreSQL metadata
echo "Exporting PostgreSQL metadata..."
pg_dump "$POSTGRES_URL" \
    --format=custom \
    --file="$EXPORT_DIR/metadata/hafiz.dump"

# Export as SQL for verification
pg_dump "$POSTGRES_URL" \
    --format=plain \
    --file="$EXPORT_DIR/metadata/hafiz.sql"

# 2. Export bucket list
echo "Exporting bucket list..."
aws --endpoint-url $S3_ENDPOINT s3 ls > "$EXPORT_DIR/metadata/buckets.txt"

# 3. Export objects for each bucket
echo "Exporting objects..."
while read -r _ _ _ bucket; do
    echo "  Bucket: $bucket"
    mkdir -p "$EXPORT_DIR/objects/$bucket"

    # Sync all objects
    aws --endpoint-url $S3_ENDPOINT s3 sync \
        "s3://$bucket" "$EXPORT_DIR/objects/$bucket/" \
        --no-progress

    # Create manifest
    find "$EXPORT_DIR/objects/$bucket" -type f -exec sha256sum {} \; \
        > "$EXPORT_DIR/checksums/$bucket.sha256"
done < "$EXPORT_DIR/metadata/buckets.txt"

# 4. Create master checksum
echo "Creating master checksums..."
find "$EXPORT_DIR" -type f -exec sha256sum {} \; \
    > "$EXPORT_DIR/CHECKSUMS.sha256"

# 5. Create export manifest
cat > "$EXPORT_DIR/MANIFEST.json" << EOF
{
  "export_type": "hafiz_airgap_export",
  "version": "1.0",
  "timestamp": "$(date -Iseconds)",
  "source_cluster": "$(hostname)",
  "postgres_version": "$(psql --version | head -1)",
  "total_buckets": $(wc -l < "$EXPORT_DIR/metadata/buckets.txt"),
  "total_size_bytes": $(du -sb "$EXPORT_DIR" | cut -f1)
}
EOF

# 6. Create archive (optional)
echo "Creating archive..."
cd "$(dirname $EXPORT_DIR)"
tar -cvf "$(basename $EXPORT_DIR).tar" "$(basename $EXPORT_DIR)"

# 7. Calculate final checksums
sha256sum "$(basename $EXPORT_DIR).tar" > "$(basename $EXPORT_DIR).tar.sha256"

echo ""
echo "=== Export Complete ==="
echo "Archive: $EXPORT_DIR.tar"
echo "Checksum: $EXPORT_DIR.tar.sha256"
echo "Size: $(du -sh $EXPORT_DIR.tar | cut -f1)"
```

#### 2. Run Export

```bash
chmod +x hafiz-export.sh
sudo ./hafiz-export.sh
```

#### 3. Transfer to Removable Media

```bash
# Mount encrypted USB drive
sudo cryptsetup luksOpen /dev/sdb1 secure_usb
sudo mount /dev/mapper/secure_usb /mnt/usb

# Copy export
cp /mnt/export/hafiz-*.tar /mnt/usb/
cp /mnt/export/hafiz-*.sha256 /mnt/usb/

# Unmount and lock
sudo umount /mnt/usb
sudo cryptsetup luksClose secure_usb
```

### Import Data to Target Cluster

#### 1. Create Import Script

```bash
#!/bin/bash
# hafiz-import.sh - Import Hafiz data from air-gapped transfer

set -e

IMPORT_FILE="$1"
POSTGRES_URL="postgresql://hafiz:password@localhost:5432/hafiz"
S3_ENDPOINT="http://localhost:9000"

if [ -z "$IMPORT_FILE" ]; then
    echo "Usage: $0 <hafiz-export.tar>"
    exit 1
fi

echo "=== Hafiz Air-Gapped Import ==="
echo "Import file: $IMPORT_FILE"
echo "Timestamp: $(date -Iseconds)"

# 1. Verify checksum
echo "Verifying archive checksum..."
if ! sha256sum -c "$IMPORT_FILE.sha256"; then
    echo "ERROR: Checksum verification failed!"
    exit 1
fi

# 2. Extract archive
IMPORT_DIR="/tmp/hafiz-import-$$"
mkdir -p "$IMPORT_DIR"
tar -xvf "$IMPORT_FILE" -C "$IMPORT_DIR"
EXPORT_DIR=$(ls "$IMPORT_DIR")

# 3. Verify internal checksums
echo "Verifying internal checksums..."
cd "$IMPORT_DIR/$EXPORT_DIR"
if ! sha256sum -c CHECKSUMS.sha256; then
    echo "ERROR: Internal checksum verification failed!"
    exit 1
fi

# 4. Import PostgreSQL metadata
echo "Importing PostgreSQL metadata..."
# Note: This overwrites existing data - backup first!
pg_restore \
    --dbname="$POSTGRES_URL" \
    --clean \
    --if-exists \
    --no-owner \
    "$IMPORT_DIR/$EXPORT_DIR/metadata/hafiz.dump"

# 5. Import objects
echo "Importing objects..."
for bucket_dir in "$IMPORT_DIR/$EXPORT_DIR/objects/"*/; do
    bucket=$(basename "$bucket_dir")
    echo "  Bucket: $bucket"

    # Create bucket if not exists
    aws --endpoint-url $S3_ENDPOINT s3 mb "s3://$bucket" 2>/dev/null || true

    # Sync objects
    aws --endpoint-url $S3_ENDPOINT s3 sync \
        "$bucket_dir" "s3://$bucket/" \
        --no-progress
done

# 6. Verify import
echo "Verifying import..."
aws --endpoint-url $S3_ENDPOINT s3 ls

# 7. Cleanup
rm -rf "$IMPORT_DIR"

echo ""
echo "=== Import Complete ==="
echo "Imported from: $IMPORT_FILE"
echo "Timestamp: $(date -Iseconds)"
```

#### 2. Run Import

```bash
# Mount USB
sudo cryptsetup luksOpen /dev/sdb1 secure_usb
sudo mount /dev/mapper/secure_usb /mnt/usb

# Verify and import
chmod +x hafiz-import.sh
sudo ./hafiz-import.sh /mnt/usb/hafiz-20240115-120000.tar

# Cleanup
sudo umount /mnt/usb
sudo cryptsetup luksClose secure_usb
```

### Incremental Air-Gapped Sync

For ongoing synchronization, export only changes since last sync:

```bash
#!/bin/bash
# hafiz-incremental-export.sh

LAST_SYNC_FILE="/var/lib/hafiz/last_airgap_sync"
LAST_SYNC=$(cat "$LAST_SYNC_FILE" 2>/dev/null || echo "1970-01-01")

EXPORT_DIR="/mnt/export/hafiz-incremental-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$EXPORT_DIR"/{metadata,objects}

# Export changed objects only
POSTGRES_URL="postgresql://hafiz:password@localhost:5432/hafiz"

# Get modified objects since last sync
psql "$POSTGRES_URL" -t -A -c "
    SELECT bucket, key FROM objects
    WHERE updated_at > '$LAST_SYNC'
    ORDER BY bucket, key
" > "$EXPORT_DIR/metadata/changed_objects.txt"

# Export changed objects
while IFS='|' read -r bucket key; do
    mkdir -p "$EXPORT_DIR/objects/$bucket/$(dirname $key)"
    aws --endpoint-url http://localhost:9000 s3 cp \
        "s3://$bucket/$key" \
        "$EXPORT_DIR/objects/$bucket/$key"
done < "$EXPORT_DIR/metadata/changed_objects.txt"

# Update last sync timestamp
date -Iseconds > "$LAST_SYNC_FILE"

# Create archive
tar -cvf "$EXPORT_DIR.tar" -C "$(dirname $EXPORT_DIR)" "$(basename $EXPORT_DIR)"
sha256sum "$EXPORT_DIR.tar" > "$EXPORT_DIR.tar.sha256"
```

### Security Considerations for Air-Gapped Systems

1. **Media Handling**
   - Use hardware-encrypted USB drives
   - Implement chain-of-custody procedures
   - Scan media for malware before import

2. **Data Integrity**
   - Always verify checksums before import
   - Use multiple checksum algorithms (SHA-256, SHA-512)
   - Keep export logs for audit

3. **Access Control**
   - Limit export/import permissions to authorized personnel
   - Log all export/import operations
   - Implement two-person rule for sensitive data

4. **Automation**
   ```bash
   # Cron job for weekly exports
   0 2 * * 0 /opt/hafiz/hafiz-export.sh >> /var/log/hafiz-export.log 2>&1
   ```

---

## Failover and Recovery

### Automatic Failover with Keepalived

```bash
# Install on load balancer nodes
sudo dnf install -y keepalived

# /etc/keepalived/keepalived.conf (Primary)
vrrp_instance HAFIZ_VIP {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass hafiz_secret
    }

    virtual_ipaddress {
        192.168.1.100/24
    }

    track_script {
        chk_haproxy
    }
}

vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
```

### Manual Failover Procedure

```bash
# 1. Stop primary cluster nodes
for node in 192.168.1.{10,11,12}; do
    ssh root@$node "docker stop hafiz"
done

# 2. Promote secondary PostgreSQL (if using streaming replication)
ssh root@10.0.1.5 "sudo -u postgres psql -c 'SELECT pg_promote()'"

# 3. Update DNS or VIP to point to secondary
# 4. Start secondary nodes if not running
# 5. Verify service
aws --endpoint-url https://hafiz-secondary.example.com s3 ls
```
