---
title: Configuration
description: Configure Hafiz for your environment
---

# Configuration

Hafiz can be configured via environment variables or configuration file.

## Environment Variables

### Required

```bash
HAFIZ_ROOT_ACCESS_KEY=your-access-key
HAFIZ_ROOT_SECRET_KEY=your-secret-key
```

### Server

```bash
HAFIZ_S3_PORT=9000          # S3 API, Admin UI, and Metrics all on this port
HAFIZ_REGION=us-east-1
HAFIZ_LOG_LEVEL=info        # trace, debug, info, warn, error
```

Access points on the S3 port:

- S3 API: `http://localhost:9000`
- Admin UI: `http://localhost:9000/admin`
- Metrics: `http://localhost:9000/metrics`
- Health: `http://localhost:9000/health`

### Storage

```bash
HAFIZ_STORAGE_TYPE=filesystem
HAFIZ_STORAGE_BASE_PATH=/data
```

### Database

```bash
# PostgreSQL (production)
HAFIZ_DATABASE_URL=postgres://user:pass@host/hafiz

# SQLite (development)
HAFIZ_METADATA_TYPE=sqlite
HAFIZ_SQLITE_PATH=/data/metadata.db
```

### Encryption

```bash
HAFIZ_ENCRYPTION_ENABLED=true
HAFIZ_ENCRYPTION_MASTER_KEY=$(openssl rand -base64 32)
```

### Cluster (multi-node)

```bash
HAFIZ_CLUSTER_ENABLED=true
HAFIZ_CLUSTER_NAME=hafiz-production
HAFIZ_CLUSTER_ADVERTISE_ENDPOINT=http://node1.example.com:9000
HAFIZ_CLUSTER_SEED_NODES=http://seed.example.com:9000    # empty on the seed itself
HAFIZ_CLUSTER_PORT=9001

# REQUIRED in production. Same value on every node. Generate with:
#   openssl rand -hex 32
HAFIZ_CLUSTER_SHARED_SECRET=<32-byte hex>
```

See [Cluster Peer Auth](../deployment/cluster-auth.md) for what the shared secret does and how to rotate it.

### Event Bridge (optional)

Set a webhook URL and every S3 mutation is POSTed as JSON:

```bash
HAFIZ_EVENT_WEBHOOK_URL=https://hooks.example.com/hafiz
HAFIZ_EVENT_WEBHOOK_AUTH_HEADER="Bearer your-token"      # optional
```

Details: [Event Bridge](../user-guide/event-bridge.md).

## Using `.env` with Docker Compose

Both shipped compose files read from `.env` in the repo root. Copy the template, fill in the blanks, start the stack:

```bash
cp .env.example .env
docker compose up -d                                     # single-node
# or
docker compose -f docker-compose.cluster.yml up -d       # cluster
```

`.env.example` lists every supported variable with inline comments. Variables marked `CHANGE_ME` must be rotated before production.

## Configuration File (alternative to env vars)

Location: `/etc/hafiz/config.toml`

```toml
[server]
bind_address = "0.0.0.0"
port = 9000

[storage]
data_dir = "/data/objects"

[database]
url = "postgresql://hafiz:secret@localhost/hafiz"

[auth]
root_access_key = "hafizadmin"
root_secret_key = "CHANGE_ME"

[encryption]
enabled = true

[cluster]
enabled = true
name = "hafiz-production"
advertise_endpoint = "http://node1:9000"
seed_nodes = ["http://seed:9000"]
cluster_port = 9001
# shared_secret = "CHANGE_ME_run_openssl_rand_hex_32"
```

See `configs/hafiz.example.toml` in the repo for the full annotated template.

## All options (env-var → TOML equivalent)

| Variable | Default | Notes |
|---|---|---|
| `HAFIZ_ROOT_ACCESS_KEY` | `hafizadmin` | Root user access key (**change in production**) |
| `HAFIZ_ROOT_SECRET_KEY` | `hafizadmin` | Root user secret key (**change in production**) |
| `HAFIZ_PORT` | 9000 | S3 API + Admin UI + Metrics share this port |
| `HAFIZ_BIND_ADDRESS` | `0.0.0.0` | Interface to bind |
| `HAFIZ_DATA_DIR` | `/data/hafiz` | Object storage directory |
| `HAFIZ_DATABASE_URL` | embedded SQLite | `postgresql://...` for cluster mode |
| `HAFIZ_LOG_LEVEL` | `info` | `trace`, `debug`, `info`, `warn`, `error` |
| `HAFIZ_TLS_CERT` / `HAFIZ_TLS_KEY` | unset | TLS certificate paths |
| `HAFIZ_TLS_CLIENT_CA` | unset | mTLS — require client certs signed by this CA |
| `HAFIZ_TLS_REQUIRE_CLIENT_CERT` | false | Enforce mTLS (fail if client cert missing) |
| `HAFIZ_IP_WHITELIST` / `HAFIZ_IP_BLACKLIST` | unset | Comma-separated IPs / CIDRs |
| `HAFIZ_RATE_LIMIT_GLOBAL_RPS` | unset | Requests/sec, all clients |
| `HAFIZ_RATE_LIMIT_PER_USER_RPS` | unset | Requests/sec, per access-key |
| `HAFIZ_RATE_LIMIT_PER_BUCKET_RPS` | unset | Requests/sec, per bucket |
| `HAFIZ_CLUSTER_ENABLED` | false | Turn on cluster mode |
| `HAFIZ_CLUSTER_NAME` | `hafiz-cluster` | Must match on every node |
| `HAFIZ_CLUSTER_ADVERTISE_ENDPOINT` | auto | This node's reachable URL |
| `HAFIZ_CLUSTER_SEED_NODES` | empty | Comma-separated seed URLs; empty = I am the seed |
| `HAFIZ_CLUSTER_PORT` | 9001 | Inter-node port |
| `HAFIZ_CLUSTER_SHARED_SECRET` | unset | **Required for authenticated peers**; HMAC-SHA256 signs every `/cluster/*` request |
| `HAFIZ_EVENT_WEBHOOK_URL` | unset | Opt-in webhook for PUT / DELETE / MPU events |
| `HAFIZ_EVENT_WEBHOOK_AUTH_HEADER` | unset | Optional `Authorization` header for outgoing webhook POSTs |
