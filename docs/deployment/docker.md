---
title: Docker Deployment
description: Running Hafiz with Docker
---

# Docker Deployment

## Building the Image

Since Hafiz is in active development, you'll need to build the Docker image locally:

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
docker build -t hafiz:local .
```

## Single Container

```bash
docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
  -e HAFIZ_ROOT_SECRET_KEY=hafizadmin \
  hafiz:local
```

Access points:

- **S3 API**: http://localhost:9000
- **Admin UI**: http://localhost:9000/admin
- **Metrics**: http://localhost:9000/metrics

## Docker Compose

### Basic Setup

```yaml
services:
  hafiz:
    build: .
    ports:
      - "9000:9000"
    volumes:
      - hafiz-data:/data
    environment:
      - HAFIZ_ROOT_ACCESS_KEY=hafizadmin
      - HAFIZ_ROOT_SECRET_KEY=hafizadmin

volumes:
  hafiz-data:
```

### Cluster Mode (3-Node with PostgreSQL)

For high availability deployments, use the included cluster configuration with shared PostgreSQL metadata, native object replication, and HMAC-authenticated peer gossip:

```bash
cp .env.example .env

# REQUIRED — cluster won't boot without this. Same value on every node.
echo "HAFIZ_CLUSTER_SHARED_SECRET=$(openssl rand -hex 32)" >> .env

# Run cluster with PostgreSQL and HAProxy load balancer
docker compose -f docker-compose.cluster.yml up -d
```

The compose file marks `HAFIZ_CLUSTER_SHARED_SECRET` as required, so you get a loud error instead of an unauthenticated cluster if you forget to set it. See [Cluster Peer Auth](cluster-auth.md) for the threat model and the rotation procedure.

This provides:

- 3 Hafiz nodes for high availability
- PostgreSQL for shared metadata (buckets, objects, users)
- **Native async object replication** across all nodes (no external tools needed)
- HAProxy load balancing with source-based consistent hashing
- Automatic replication of puts, deletes, and multipart uploads

Access points:
- **S3 API (direct nodes)**: http://localhost:9000, :9010, :9020
- **S3 API (load balanced)**: http://localhost:80
- **HAProxy Stats**: http://localhost:8404/stats
- **PostgreSQL**: localhost:5432

### Distributed Cluster (Physical Servers)

For deploying across separate physical servers, see `deploy/distributed/`:

```bash
# Node1: PostgreSQL + HAProxy + Hafiz
docker compose -f deploy/distributed/node1-compose.yml up -d

# Node2: Hafiz only (connects to Node1's PostgreSQL)
docker compose -f deploy/distributed/node2-compose.yml up -d

# Node3: Hafiz only (connects to Node1's PostgreSQL)
docker compose -f deploy/distributed/node3-compose.yml up -d
```

Objects uploaded to any node are automatically replicated to all other healthy nodes. See [deploy/distributed/README.md](../../deploy/distributed/README.md) for full details.

### Running Tests

```bash
# S3 API tests (52 tests) - works on single-node or cluster
export AWS_ACCESS_KEY_ID=hafizadmin
export AWS_SECRET_ACCESS_KEY='HafizSecret2024!'
bash deploy/distributed/tests/run-tests.sh

# Replication tests (29 tests) - requires 3-node cluster
bash deploy/distributed/tests/replication-tests.sh
```

**Verified: 52/52 S3 API + 29/29 Replication = 81/81 PASS**

## Environment Variables

The shipped `docker-compose.yml` / `docker-compose.cluster.yml` read from `.env` in the repo root. Copy `.env.example` and fill it in — every variable that needs attention before production is marked `CHANGE_ME`.

| Variable | Required | Default | Description |
|---|---|---|---|
| `HAFIZ_ROOT_ACCESS_KEY` | yes | `hafizadmin` | Root access key — **change before production** |
| `HAFIZ_ROOT_SECRET_KEY` | yes | `hafizadmin` | Root secret key — **change before production** |
| `HAFIZ_PORT` | no | 9000 | S3 API / Admin UI / Metrics port |
| `HAFIZ_DATA_DIR` | no | `/data/objects` | Object storage directory inside the container |
| `HAFIZ_DATABASE_URL` | no | embedded SQLite | Switch to `postgresql://...` in cluster mode |
| `HAFIZ_LOG_LEVEL` | no | `info` | `trace` / `debug` / `info` / `warn` / `error` |
| `HAFIZ_CLUSTER_SHARED_SECRET` | cluster: yes | unset | HMAC key signing every `/cluster/*` request; same on every node. Generate with `openssl rand -hex 32`. |
| `HAFIZ_EVENT_WEBHOOK_URL` | no | unset | Opt-in webhook — receives PUT / DELETE / MPU events as JSON. |
| `HAFIZ_EVENT_WEBHOOK_AUTH_HEADER` | no | unset | `Authorization` header appended to every outgoing webhook POST. |
| `POSTGRES_PASSWORD` | cluster: yes | `hafizpassword` | PostgreSQL password (cluster mode). |

Full list with operational details: [Configuration](../getting-started/configuration.md#all-options-env-var--toml-equivalent).

## Verify Installation

```bash
# Check metrics (confirms server is running)
curl http://localhost:9000/metrics

# Test S3 API - list buckets
curl http://localhost:9000/

# Test with AWS CLI
aws --endpoint-url http://localhost:9000 s3 ls

# Access Admin UI
open http://localhost:9000/admin
```

## Troubleshooting

### Container won't start

Check logs:

```bash
docker logs hafiz
```

### Permission issues

Ensure the data volume is writable:

```bash
docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  -u $(id -u):$(id -g) \
  hafiz:local
```

### Health check failing

Wait for the server to fully start:

```bash
docker logs -f hafiz
# Wait for "Server started on port 9000"
```
