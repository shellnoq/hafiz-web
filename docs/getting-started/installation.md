---
title: Installation
description: Install Hafiz on your system
---

# Installation

Two supported install paths. Both start from a fresh clone and the shipped `.env` template — pick the one that matches what you're standing up.

## Single-node (simplest)

Good for dev, evaluation, small-scale production. Uses embedded SQLite — zero external dependencies.

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
cp .env.example .env                                    # edit, set HAFIZ_ROOT_SECRET_KEY
docker compose up -d
```

That's everything. Compose builds the image on first run, starts the server on port 9000, and mounts a named volume for data.

```
S3 API:   http://localhost:9000
Admin UI: http://localhost:9000/admin
Metrics:  http://localhost:9000/metrics
```

## Cluster (3 nodes + PostgreSQL + HAProxy)

For HA, replication, and peer-authenticated multi-node setups.

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
cp .env.example .env

# REQUIRED — cluster won't boot without this. Same value on every node.
echo "HAFIZ_CLUSTER_SHARED_SECRET=$(openssl rand -hex 32)" >> .env

docker compose -f docker-compose.cluster.yml up -d
```

Access via HAProxy on `http://localhost`, or directly to node 1/2/3 on `9000`/`9010`/`9020`.

!!! note "Why a shared secret is required"
    Every `/cluster/*` peer request is HMAC-SHA256 signed. Without the secret the compose file fails fast instead of booting an unauthenticated cluster. See [Cluster Peer Auth](../deployment/cluster-auth.md) for the threat model and rotation procedure.

## Run a single container by hand

If you don't want compose at all:

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
docker build -t hafiz:local .

docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
  -e HAFIZ_ROOT_SECRET_KEY=CHANGE_ME \
  hafiz:local
```

## Binary Downloads

Pre-built binaries will be available from [GitHub Releases](https://github.com/shellnoq/hafiz/releases) after the first stable release:

| Platform | Download |
|----------|----------|
| Linux (amd64) | `hafiz-linux-amd64.tar.gz` |
| Linux (arm64) | `hafiz-linux-arm64.tar.gz` |
| macOS (amd64) | `hafiz-darwin-amd64.tar.gz` |
| macOS (arm64) | `hafiz-darwin-arm64.tar.gz` |
| Windows | `hafiz-windows-amd64.zip` |

```bash
# Linux/macOS (when available)
curl -LO https://github.com/shellnoq/hafiz/releases/latest/download/hafiz-linux-amd64.tar.gz
tar xzf hafiz-linux-amd64.tar.gz
sudo mv hafiz-server /usr/local/bin/
```

## From Source

### Prerequisites

- Rust 1.85+
- PostgreSQL 13+ (optional, for cluster mode)

### Build

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
cargo build --release
```

### Install

```bash
sudo cp target/release/hafiz-server /usr/local/bin/
sudo cp target/release/hafiz /usr/local/bin/
```

### Run

```bash
HAFIZ_ROOT_ACCESS_KEY=admin \
HAFIZ_ROOT_SECRET_KEY=password \
./target/release/hafiz-server
```

## Verify

```bash
# Check version
hafiz-server --version

# Check health endpoint
curl http://localhost:9000/health

# Access Admin UI
open http://localhost:9000/admin
```
