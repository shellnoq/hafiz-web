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

For high availability deployments, use the included cluster configuration with shared PostgreSQL metadata and native object replication:

```bash
# Build image first
docker build -t hafiz:latest .

# Run cluster with PostgreSQL and HAProxy load balancer
docker compose -f docker-compose.cluster.yml up -d
```

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

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `HAFIZ_ROOT_ACCESS_KEY` | Yes | - | Root access key |
| `HAFIZ_ROOT_SECRET_KEY` | Yes | - | Root secret key |
| `HAFIZ_S3_PORT` | No | 9000 | S3 API port |
| `HAFIZ_STORAGE_BASE_PATH` | No | /data/hafiz | Data directory |
| `HAFIZ_DATABASE_URL` | No | sqlite:///data/hafiz/hafiz.db | Database URL (SQLite or PostgreSQL) |
| `HAFIZ_LOG_LEVEL` | No | info | Log level |
| `HAFIZ_ENCRYPTION_ENABLED` | No | false | Enable SSE |
| `POSTGRES_PASSWORD` | No | hafizpassword | PostgreSQL password (cluster mode) |

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
