---
title: Multi-Server Cluster Deployment
description: Deploy Hafiz across multiple physical servers
---

# Multi-Server Cluster Deployment

This guide covers deploying Hafiz across multiple physical servers with shared PostgreSQL metadata.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Quick Start: Adding a Second Node (Native/Bare Metal)](#quick-start-adding-a-second-node-nativebare-metal)
- [Quick Start: Dual Cluster with Docker Compose](#quick-start-dual-cluster-with-docker-compose)
- [Single-Network Cluster](#step-1-postgresql-setup)
- [Adding Servers to Existing Cluster](#adding-servers-to-existing-cluster)
- [Cross-Network Replication](#cross-network-replication)
- [What is Air-Gap?](#what-is-air-gap)
- [Unidirectional Replication (One-Way Sync)](#unidirectional-replication-one-way-sync)
- [Air-Gapped System Replication](#air-gapped-system-replication)
- [Failover and Recovery](#failover-and-recovery)
- [Node Management API](#node-management-api)

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

---

## Quick Start: Adding a Second Node (Native/Bare Metal)

This section covers the simplest cluster setup: adding a second Hafiz node to an existing primary node, using native binaries (no Docker) and SQLite for metadata.

### Two-Node Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   ┌─────────────────┐                 ┌─────────────────┐           │
│   │   Primary Node  │                 │  Secondary Node │           │
│   │  (Read/Write)   │ ──Replication──▶│    (Replica)    │           │
│   │                 │                 │                 │           │
│   │ dev-hafiz.e2e.lab:9000           │ dev-hafiz-node1.e2e.lab:9000│
│   │ SQLite (local)  │                 │ SQLite (local)  │           │
│   └─────────────────┘                 └─────────────────┘           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Prerequisites

- Primary Hafiz node already running
- Secondary server with:
  - Linux (Rocky Linux 8/9, RHEL 8/9, Ubuntu 20.04+)
  - Network connectivity to primary node
  - Port 9000 accessible (firewall opened)
  - DNS A record pointing to the server (recommended)

### Step 1: Copy the Binary to Secondary Node

On the **primary node**, copy the hafiz-server binary:

```bash
# From primary node
scp /opt/hafiz/target/release/hafiz-server user@secondary-node:/opt/hafiz/
```

Or on the **secondary node**, pull from primary:

```bash
# On secondary node
mkdir -p /opt/hafiz
scp user@primary-node:/opt/hafiz/target/release/hafiz-server /opt/hafiz/
chmod +x /opt/hafiz/hafiz-server
```

### Step 2: Run the Setup Script

Hafiz includes a setup script for secondary nodes:

```bash
# Download and customize the script
curl -O https://raw.githubusercontent.com/shellnoq/hafiz/main/scripts/setup-node2.sh

# Edit configuration variables at the top of the script
vim setup-node2.sh

# Run the script
chmod +x setup-node2.sh
sudo ./setup-node2.sh
```

The script will:
1. Create required directories
2. Generate the configuration file
3. Create a systemd service
4. Configure the firewall
5. Optionally start the service

### Step 3: Manual Setup (Alternative)

If you prefer manual setup instead of using the script:

#### 3a. Create Configuration File

```bash
mkdir -p /opt/hafiz/data

cat > /opt/hafiz/data/hafiz.toml << 'EOF'
[server]
bind_address = "0.0.0.0"
port = 9000
admin_port = 9001
workers = 0
max_connections = 10000
request_timeout_secs = 300

[storage]
data_dir = "/opt/hafiz/data"
temp_dir = "/tmp/hafiz"
max_object_size = 5497558138880

[database]
url = "sqlite:///opt/hafiz/data/hafiz.db?mode=rwc"
max_connections = 100
min_connections = 5

[auth]
enabled = true
root_access_key = "minioadmin"
root_secret_key = "minioadmin"

[logging]
level = "info"
format = "pretty"

[cluster]
enabled = true
name = "hafiz-cluster"
advertise_endpoint = "http://your-secondary-node.example.com:9000"
cluster_port = 9001
seed_nodes = ["http://your-primary-node.example.com:9000"]
heartbeat_interval_secs = 5
node_timeout_secs = 30
default_replication_mode = "async"
default_replication_factor = 2
cluster_tls_enabled = false
EOF
```

#### 3b. Create Systemd Service

```bash
cat > /etc/systemd/system/hafiz.service << 'EOF'
[Unit]
Description=Hafiz S3 Storage Server
After=network.target
Documentation=https://github.com/shellnoq/hafiz

[Service]
Type=simple
User=your-user
Group=your-group
Environment="HAFIZ_CONFIG_FILE=/opt/hafiz/data/hafiz.toml"
ExecStart=/opt/hafiz/hafiz-server
Restart=always
RestartSec=5
WorkingDirectory=/opt/hafiz

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

#### 3c. Open Firewall Ports

```bash
sudo firewall-cmd --add-port=9000/tcp --permanent
sudo firewall-cmd --add-port=9001/tcp --permanent
sudo firewall-cmd --reload
```

#### 3d. Start the Service

```bash
sudo systemctl start hafiz
sudo systemctl enable hafiz
sudo systemctl status hafiz
```

### Step 4: Register Node with Primary Cluster

After the secondary node is running, register it with the primary:

```bash
curl -X POST http://primary-node:9000/api/v1/cluster/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "http://secondary-node:9000",
    "name": "secondary-node",
    "role": "replica"
  }'
```

### Step 5: Create Replication Rules

Set up replication for your buckets:

```bash
# Create a replication rule for a bucket
curl -X POST http://primary-node:9000/api/v1/cluster/replication/rules \
  -H "Content-Type: application/json" \
  -d '{
    "source_bucket": "my-bucket",
    "destination_endpoint": "http://secondary-node:9000",
    "mode": "async"
  }'
```

### Step 6: Verify Cluster and Replication

```bash
# Check cluster nodes
curl -s http://primary-node:9000/api/v1/cluster/nodes | jq .

# Check replication stats
curl -s http://primary-node:9000/api/v1/cluster/replication/stats | jq .

# Test replication - upload to primary
curl -X PUT http://primary-node:9000/my-bucket/test.txt -d "Hello World"

# Wait a moment, then verify on secondary
curl http://secondary-node:9000/my-bucket/test.txt
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| Connection refused to secondary | Check firewall: `firewall-cmd --list-ports` |
| Config parse error on start | Ensure all required fields in hafiz.toml (see example config) |
| Permission denied on start | Check file ownership matches User in systemd service |
| Replication not working | Ensure bucket exists on secondary: `curl -X PUT http://secondary:9000/bucket-name` |
| Node not showing in cluster | Register manually via API (Step 4 above) |

### Useful Commands

```bash
# View service logs
sudo journalctl -u hafiz -f

# Restart service
sudo systemctl restart hafiz

# Check if service is enabled for boot
sudo systemctl is-enabled hafiz

# Test connectivity from primary to secondary
curl http://secondary-node:9000/

# View cluster status
curl http://primary-node:9000/api/v1/cluster/status | jq .
```

---

## Prerequisites

- 3+ servers with Rocky Linux 8/9 or RHEL 8/9
- PostgreSQL 13+ on a dedicated server (or managed service)
- Network connectivity between all nodes
- Domain name with DNS configured (e.g., `dev-hafiz.e2e.lab`)

---

## Quick Start: Dual Cluster with Docker Compose

This section provides a complete step-by-step guide for deploying two Hafiz clusters with automatic synchronization.

### Dual Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CLUSTER A (Primary)                               │
│                          Server: 192.168.1.100                               │
│                                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────────┐   │
│   │ hafiz-node1 │   │ hafiz-node2 │   │ hafiz-node3 │   │  PostgreSQL  │   │
│   │   :9000     │   │   :9010     │   │   :9020     │   │    :5432     │   │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬───────┘   │
│          └─────────────────┴─────────────────┴─────────────────┘            │
│                                      │                                       │
│                              ┌───────┴───────┐                              │
│                              │   HAProxy     │                              │
│                              │   :80/:443    │                              │
│                              └───────────────┘                              │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                           ┌──────────┴──────────┐
                           │  Network / VPN /    │
                           │   Internet Link     │
                           └──────────┬──────────┘
                                      │
┌─────────────────────────────────────┴───────────────────────────────────────┐
│                            CLUSTER B (Secondary)                             │
│                          Server: 192.168.2.100                               │
│                                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────────┐   │
│   │ hafiz-node1 │   │ hafiz-node2 │   │ hafiz-node3 │   │  PostgreSQL  │   │
│   │   :9000     │   │   :9010     │   │   :9020     │   │    :5432     │   │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬───────┘   │
│          └─────────────────┴─────────────────┴─────────────────┘            │
│                                      │                                       │
│                              ┌───────┴───────┐                              │
│                              │   HAProxy     │                              │
│                              │   :80/:443    │                              │
│                              └───────────────┘                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step 1: Deploy Cluster A (Primary)

On the first server (192.168.1.100):

```bash
# Clone the repository
git clone https://github.com/shellnoq/hafiz.git
cd hafiz

# Create environment file
cat > .env << 'EOF'
POSTGRES_PASSWORD=cluster_a_password_here
HAFIZ_ROOT_ACCESS_KEY=hafizadmin
HAFIZ_ROOT_SECRET_KEY=your_secret_key_here
EOF

# Build the Docker image
docker build -t hafiz:latest .

# Start Cluster A
docker compose -f docker-compose.cluster.yml up -d

# Verify all containers are running
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE                COMMAND                  STATUS          PORTS
xxxx           hafiz:latest         "/usr/bin/tini -- ha…"   Up (healthy)    0.0.0.0:9000->9000/tcp
xxxx           hafiz:latest         "/usr/bin/tini -- ha…"   Up (healthy)    0.0.0.0:9010->9000/tcp
xxxx           hafiz:latest         "/usr/bin/tini -- ha…"   Up (healthy)    0.0.0.0:9020->9000/tcp
xxxx           postgres:16-alpine   "docker-entrypoint.s…"   Up (healthy)    0.0.0.0:5432->5432/tcp
xxxx           haproxy:2.9-alpine   "docker-entrypoint.s…"   Up              0.0.0.0:80->80/tcp
```

### Step 2: Verify Cluster A is Working

```bash
# Check health endpoint
curl http://localhost:9000/health

# Access admin panel
echo "Admin Panel: http://192.168.1.100:9000/admin"

# Create a test bucket
aws --endpoint-url http://localhost:9000 s3 mb s3://test-bucket

# Upload a test file
echo "Hello from Cluster A" > test.txt
aws --endpoint-url http://localhost:9000 s3 cp test.txt s3://test-bucket/

# List objects
aws --endpoint-url http://localhost:9000 s3 ls s3://test-bucket/
```

### Step 3: Deploy Cluster B (Secondary)

On the second server (192.168.2.100):

```bash
# Clone the repository
git clone https://github.com/shellnoq/hafiz.git
cd hafiz

# Create environment file with DIFFERENT credentials
cat > .env << 'EOF'
POSTGRES_PASSWORD=cluster_b_password_here
HAFIZ_ROOT_ACCESS_KEY=hafizadmin
HAFIZ_ROOT_SECRET_KEY=your_secret_key_here
EOF

# Build the Docker image
docker build -t hafiz:latest .

# Start Cluster B
docker compose -f docker-compose.cluster.yml up -d

# Verify all containers are running
docker ps
```

### Step 4: Configure Cluster Synchronization

There are three options for synchronizing data between clusters:

#### Option A: PostgreSQL Logical Replication (Recommended for Metadata)

This keeps metadata (buckets, users, policies) synchronized in real-time.

**On Cluster A (Primary) - PostgreSQL Container:**

```bash
# Connect to PostgreSQL
docker exec -it hafiz-postgres psql -U hafiz -d hafiz

# Enable logical replication
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 4;
ALTER SYSTEM SET max_wal_senders = 4;

# Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;

# Create publication for all tables
CREATE PUBLICATION hafiz_replication FOR ALL TABLES;

# Exit and restart PostgreSQL
\q
```

```bash
docker restart hafiz-postgres
```

**On Cluster B (Secondary) - PostgreSQL Container:**

```bash
# Connect to PostgreSQL
docker exec -it hafiz-postgres psql -U hafiz -d hafiz

# Create subscription to Cluster A
CREATE SUBSCRIPTION hafiz_subscription
    CONNECTION 'host=192.168.1.100 port=5432 dbname=hafiz user=replicator password=repl_secure_password'
    PUBLICATION hafiz_replication;

# Verify subscription is active
SELECT * FROM pg_stat_subscription;

\q
```

#### Option B: Object Data Synchronization with rclone

For synchronizing actual object files between clusters:

```bash
# Install rclone on both servers
curl https://rclone.org/install.sh | sudo bash

# Configure Cluster A as source
rclone config create cluster_a s3 \
    provider=Other \
    endpoint=http://192.168.1.100:9000 \
    access_key_id=hafizadmin \
    secret_access_key=your_secret_key_here

# Configure Cluster B as target
rclone config create cluster_b s3 \
    provider=Other \
    endpoint=http://192.168.2.100:9000 \
    access_key_id=hafizadmin \
    secret_access_key=your_secret_key_here

# Test sync (dry run first)
rclone sync cluster_a:test-bucket cluster_b:test-bucket --dry-run

# Run actual sync
rclone sync cluster_a:test-bucket cluster_b:test-bucket --progress

# Verify on Cluster B
aws --endpoint-url http://192.168.2.100:9000 s3 ls s3://test-bucket/
```

**Automated Sync with Cron:**

```bash
# Create sync script
cat > /opt/hafiz/sync-clusters.sh << 'EOF'
#!/bin/bash
# Sync all buckets from Cluster A to Cluster B

LOG_FILE="/var/log/hafiz-sync.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] Starting cluster sync..." >> $LOG_FILE

# Get list of buckets from Cluster A
BUCKETS=$(rclone lsd cluster_a: | awk '{print $5}')

for bucket in $BUCKETS; do
    echo "[$TIMESTAMP] Syncing bucket: $bucket" >> $LOG_FILE
    rclone sync cluster_a:$bucket cluster_b:$bucket \
        --checksum \
        --transfers 4 \
        --checkers 8 \
        2>> $LOG_FILE
done

echo "[$TIMESTAMP] Sync completed." >> $LOG_FILE
EOF

chmod +x /opt/hafiz/sync-clusters.sh

# Add to crontab (every 5 minutes)
(crontab -l 2>/dev/null; echo "*/5 * * * * /opt/hafiz/sync-clusters.sh") | crontab -
```

#### Option C: Real-Time Bidirectional Sync

For active-active clusters where both can accept writes:

```bash
# Create bidirectional sync script
cat > /opt/hafiz/bidirectional-sync.sh << 'EOF'
#!/bin/bash
# Bidirectional sync between clusters

LOG_FILE="/var/log/hafiz-bisync.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] Starting bidirectional sync..." >> $LOG_FILE

# Get all buckets from both clusters
BUCKETS_A=$(rclone lsd cluster_a: 2>/dev/null | awk '{print $5}')
BUCKETS_B=$(rclone lsd cluster_b: 2>/dev/null | awk '{print $5}')
ALL_BUCKETS=$(echo -e "$BUCKETS_A\n$BUCKETS_B" | sort -u)

for bucket in $ALL_BUCKETS; do
    echo "[$TIMESTAMP] Bidirectional sync: $bucket" >> $LOG_FILE

    # Sync A -> B (new files from A)
    rclone copy cluster_a:$bucket cluster_b:$bucket \
        --checksum --update 2>> $LOG_FILE

    # Sync B -> A (new files from B)
    rclone copy cluster_b:$bucket cluster_a:$bucket \
        --checksum --update 2>> $LOG_FILE
done

echo "[$TIMESTAMP] Bidirectional sync completed." >> $LOG_FILE
EOF

chmod +x /opt/hafiz/bidirectional-sync.sh

# Run every minute for near real-time sync
(crontab -l 2>/dev/null; echo "* * * * * /opt/hafiz/bidirectional-sync.sh") | crontab -
```

### Step 5: Verify Synchronization

**Test metadata sync (if using PostgreSQL replication):**

```bash
# Create user on Cluster A
curl -X POST http://192.168.1.100:9000/api/v1/users \
    -H "Content-Type: application/json" \
    -d '{"name": "testuser", "email": "test@example.com"}'

# Verify user appears on Cluster B (should sync within seconds)
curl http://192.168.2.100:9000/api/v1/users
```

**Test object sync:**

```bash
# Upload to Cluster A
aws --endpoint-url http://192.168.1.100:9000 s3 cp myfile.txt s3://test-bucket/

# Wait for sync (based on your cron interval)
sleep 60

# Verify on Cluster B
aws --endpoint-url http://192.168.2.100:9000 s3 ls s3://test-bucket/
aws --endpoint-url http://192.168.2.100:9000 s3 cp s3://test-bucket/myfile.txt -
```

### Step 6: Set Up Load Balancer for Failover

Configure HAProxy or nginx to route traffic with automatic failover:

```haproxy
# /etc/haproxy/haproxy-global.cfg

global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httpchk GET /health

frontend hafiz_frontend
    bind *:9000
    default_backend hafiz_clusters

backend hafiz_clusters
    balance roundrobin
    option httpchk GET /health

    # Cluster A nodes (primary)
    server cluster_a_node1 192.168.1.100:9000 check weight 100
    server cluster_a_node2 192.168.1.100:9010 check weight 100

    # Cluster B nodes (backup)
    server cluster_b_node1 192.168.2.100:9000 check backup
    server cluster_b_node2 192.168.2.100:9010 check backup
```

### Step 7: Monitoring and Health Checks

**Create health check script:**

```bash
cat > /opt/hafiz/health-check.sh << 'EOF'
#!/bin/bash

CLUSTERS=("192.168.1.100:9000" "192.168.2.100:9000")
WEBHOOK_URL="https://your-slack-webhook-url"

for cluster in "${CLUSTERS[@]}"; do
    STATUS=$(curl -sf "http://$cluster/health" | jq -r '.status' 2>/dev/null)

    if [ "$STATUS" != "healthy" ]; then
        echo "ALERT: Cluster $cluster is unhealthy!"

        # Send alert (Slack example)
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"⚠️ Hafiz cluster $cluster is unhealthy!\"}" \
            $WEBHOOK_URL
    fi
done
EOF

chmod +x /opt/hafiz/health-check.sh

# Check every minute
(crontab -l 2>/dev/null; echo "* * * * * /opt/hafiz/health-check.sh") | crontab -
```

### Troubleshooting Dual Cluster Setup

| Issue | Solution |
|-------|----------|
| PostgreSQL replication lag | Check network latency, increase `wal_sender_timeout` |
| Objects not syncing | Verify rclone config with `rclone lsd cluster_a:` |
| Connection refused | Check firewall rules: ports 5432, 9000 |
| Authentication failed | Verify credentials in .env files match |
| Containers unhealthy | Check logs: `docker logs hafiz-node1` |

**View sync logs:**
```bash
tail -f /var/log/hafiz-sync.log
```

**Check PostgreSQL replication status:**
```bash
docker exec hafiz-postgres psql -U hafiz -d hafiz -c "SELECT * FROM pg_stat_subscription;"
```

---

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

## What is Air-Gap?

An **air-gapped network** is a security measure where a computer or network is physically isolated from other networks, including the internet. There is no wired or wireless connection between the air-gapped system and any other network.

### Why Use Air-Gap?

| Use Case | Description |
|----------|-------------|
| **Classified Networks** | Military, intelligence, and government systems handling sensitive data |
| **Critical Infrastructure** | Power grids, water treatment, nuclear facilities |
| **Financial Systems** | High-security trading systems, core banking |
| **Healthcare** | Patient data isolation, medical device networks |
| **Research Labs** | Protecting intellectual property and sensitive research |
| **Disaster Recovery** | Offline backup sites immune to ransomware attacks |

### How Hafiz Supports Air-Gap

Hafiz provides complete air-gap support through:

1. **Export/Import Tools**: Scripts to export all data (metadata + objects) to physical media
2. **Checksum Verification**: SHA-256 checksums at every level for data integrity
3. **Incremental Sync**: Export only changed objects since last sync
4. **Encrypted Media Support**: Works with LUKS-encrypted USB drives
5. **Audit Trail**: Complete logging of all export/import operations

### Air-Gap Workflow Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AIR-GAP WORKFLOW                                │
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐│
│  │   Source     │───▶│   Export     │───▶│   Physical   │───▶│   Import   ││
│  │   Cluster    │    │   Server     │    │   Transfer   │    │   Server   ││
│  │ (Read-Write) │    │              │    │  (USB/Tape)  │    │            ││
│  └──────────────┘    └──────────────┘    └──────────────┘    └─────┬──────┘│
│                                                                      │       │
│                                                                      ▼       │
│                                                              ┌────────────┐ │
│                                                              │   Target   │ │
│                                                              │   Cluster  │ │
│                                                              │ (Read-Only)│ │
│                                                              └────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Unidirectional Replication (One-Way Sync)

Hafiz supports unidirectional replication where one cluster is read-write (primary) and the other is read-only (replica). This is ideal for:

- **Disaster Recovery**: Main site writes, DR site receives copies
- **Content Distribution**: Central site publishes, edge sites consume
- **Regulatory Compliance**: Write to secure location, replicate to reporting systems
- **Air-Gapped Backup**: Secure network writes, isolated network receives

### Replication Direction Modes

| Mode | Source | Destination | Use Case |
|------|--------|-------------|----------|
| `Bidirectional` | Read-Write | Read-Write | Active-active clusters |
| `SourceToDestination` | Read-Write | Read-Only | Primary/replica setup |
| `DestinationToSource` | Read-Only | Read-Write | Reverse flow setup |

### Configuring Unidirectional Replication via API

#### Create a One-Way Replication Rule

```bash
# Create replication rule: Primary → Replica (one-way)
curl -X POST http://localhost:9000/admin/cluster/replication/rules \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "primary-to-dr",
    "source_bucket": "production-data",
    "destination_bucket": "production-data",
    "destination_endpoint": "http://dr-site.example.com:9000",
    "destination_access_key": "dr_access_key",
    "destination_secret_key": "dr_secret_key",
    "direction": "SourceToDestination",
    "status": "enabled",
    "priority": 1,
    "filter_prefix": ""
  }'
```

#### Check Replication Rule

```bash
curl -X GET http://localhost:9000/admin/cluster/replication/rules/primary-to-dr \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)"
```

Response:
```json
{
  "id": "primary-to-dr",
  "source_bucket": "production-data",
  "destination_bucket": "production-data",
  "destination_endpoint": "http://dr-site.example.com:9000",
  "direction": "SourceToDestination",
  "status": "enabled",
  "priority": 1
}
```

### Configuring Read-Only Replica

On the destination cluster, configure the bucket to reject writes:

#### Option 1: User-Level Permission (Recommended)

Create users on the replica site with read-only bucket access:

```bash
# Create read-only user for replica bucket
curl -X POST http://dr-site:9000/admin/users \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "replica-reader",
    "description": "Read-only user for replicated data",
    "bucket_access": [
      {"bucket": "production-data", "permission": "read"}
    ]
  }'
```

Response:
```json
{
  "name": "replica-reader",
  "access_key": "AKIAXXXXXXXXXXXXXXXX",
  "secret_key": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "description": "Read-only user for replicated data",
  "bucket_access": [
    {"bucket": "production-data", "permission": "read"}
  ]
}
```

#### Option 2: Replication Service Account

The replication service uses a dedicated account with write access, while all other users have read-only access:

```bash
# Replication service account (has write for sync)
curl -X POST http://dr-site:9000/admin/users \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "replication-service",
    "description": "Internal replication service - write access for sync only",
    "bucket_access": [
      {"bucket": "production-data", "permission": "write"}
    ]
  }'

# Regular users get read-only
curl -X POST http://dr-site:9000/admin/users \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "app-user",
    "description": "Application user - read only on DR site",
    "bucket_access": [
      {"bucket": "production-data", "permission": "read"}
    ]
  }'
```

### Architecture: Unidirectional Replication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PRIMARY SITE (Read-Write)                              │
│                                                                              │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                  │
│   │  App Server │────▶│   Hafiz     │────▶│ PostgreSQL  │                  │
│   │  (Writes)   │     │  Primary    │     │  Primary    │                  │
│   └─────────────┘     └──────┬──────┘     └─────────────┘                  │
│                              │                                               │
│                              │ Replication Events                            │
│                              ▼                                               │
│                     ┌────────────────┐                                      │
│                     │   Replication  │                                      │
│                     │    Service     │                                      │
│                     │ (direction:    │                                      │
│                     │  source→dest)  │                                      │
│                     └────────┬───────┘                                      │
└──────────────────────────────┼──────────────────────────────────────────────┘
                               │
                    Objects + Metadata
                               │
                               ▼
┌──────────────────────────────┼──────────────────────────────────────────────┐
│                              │                                               │
│                     ┌────────▼───────┐                                      │
│                     │   Replication  │                                      │
│                     │    Receiver    │                                      │
│                     │  (write-only   │                                      │
│                     │   service)     │                                      │
│                     └────────┬───────┘                                      │
│                              │                                               │
│   ┌─────────────┐     ┌──────▼──────┐     ┌─────────────┐                  │
│   │  App Server │────▶│   Hafiz     │────▶│ PostgreSQL  │                  │
│   │ (Read Only) │     │   Replica   │     │   Replica   │                  │
│   └─────────────┘     └─────────────┘     └─────────────┘                  │
│                                                                              │
│                      REPLICA SITE (Read-Only for Users)                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Monitoring Unidirectional Replication

```bash
# Check replication lag
curl http://primary:9000/admin/cluster/replication/stats \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)"
```

Response:
```json
{
  "events_processed": 15420,
  "successful": 15418,
  "failed": 2,
  "pending": 5,
  "in_progress": 1,
  "bytes_replicated": 1073741824,
  "avg_latency_ms": 45.2,
  "rules": [
    {
      "id": "primary-to-dr",
      "direction": "SourceToDestination",
      "pending_events": 5,
      "last_sync": "2024-01-15T12:30:00Z"
    }
  ]
}
```

### Failover to Read-Only Replica

If you need to promote the replica to primary:

```bash
# 1. Stop replication rule
curl -X DELETE http://primary:9000/admin/cluster/replication/rules/primary-to-dr \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)"

# 2. Update user permissions on replica to allow writes
curl -X PUT http://dr-site:9000/admin/users/app-user/buckets \
  -H "Authorization: Basic $(echo -n 'hafizadmin:secret' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "bucket_access": [
      {"bucket": "production-data", "permission": "readwrite"}
    ]
  }'

# 3. Update DNS to point to replica
# 4. Start accepting traffic on replica
```

---

## Air-Gapped System Replication

For environments with no network connectivity (classified networks, secure facilities, disaster recovery sites), Hafiz supports offline data transfer. See [What is Air-Gap?](#what-is-air-gap) for more context.

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

---

## Node Management API

Hafiz provides REST APIs for managing cluster nodes programmatically.

### Get Cluster Status

```bash
curl -X GET http://localhost:9000/api/v1/cluster/status \
  -H "Authorization: Bearer <token>"
```

Response:
```json
{
  "enabled": true,
  "cluster_name": "production",
  "local_node": {
    "id": "node-1",
    "name": "Node 1",
    "endpoint": "http://192.168.1.10:9000",
    "role": "primary",
    "status": "healthy"
  },
  "stats": {
    "total_nodes": 3,
    "healthy_nodes": 3,
    "total_objects": 15420,
    "total_storage_bytes": 1073741824,
    "pending_replications": 0,
    "replication_lag_secs": 0
  }
}
```

### List Cluster Nodes

```bash
curl -X GET http://localhost:9000/api/v1/cluster/nodes \
  -H "Authorization: Bearer <token>"
```

### Drain a Node (Maintenance Mode)

Draining a node gracefully stops it from accepting new writes and completes pending replications:

```bash
curl -X POST http://localhost:9000/api/v1/cluster/nodes/<node-id>/drain \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"graceful": true, "timeout_secs": 300}'
```

Response:
```json
{
  "status": "draining",
  "node_id": "node-2",
  "message": "Node drain initiated. The node will stop accepting writes and finish pending replications."
}
```

### Remove a Node from Cluster

Remove a node permanently from the cluster:

```bash
curl -X DELETE http://localhost:9000/api/v1/cluster/nodes/<node-id> \
  -H "Authorization: Bearer <token>"
```

Response:
```json
{
  "status": "removed",
  "node_id": "node-2",
  "message": "Node removed from cluster. Data rebalancing may be needed if the node held unique data."
}
```

### Replication Statistics

```bash
curl -X GET http://localhost:9000/api/v1/cluster/replication/stats \
  -H "Authorization: Bearer <token>"
```

Response:
```json
{
  "events_processed": 15420,
  "successful": 15418,
  "failed": 2,
  "pending": 0,
  "in_progress": 0,
  "bytes_replicated": 1073741824,
  "avg_latency_ms": 12.5
}
```

### Maintenance Workflow

For planned maintenance on a node:

```bash
# 1. Drain the node (stop new writes, complete pending work)
curl -X POST http://localhost:9000/api/v1/cluster/nodes/node-2/drain \
  -H "Authorization: Bearer <token>" \
  -d '{"graceful": true}'

# 2. Wait for drain to complete (check status)
curl -X GET http://localhost:9000/api/v1/cluster/nodes/node-2 \
  -H "Authorization: Bearer <token>"
# Check that status is "draining" or "drained"

# 3. Perform maintenance on the node
ssh root@node-2 "systemctl stop hafiz && dnf update -y && systemctl start hafiz"

# 4. Verify node rejoins cluster
curl -X GET http://localhost:9000/api/v1/cluster/nodes \
  -H "Authorization: Bearer <token>"
```
