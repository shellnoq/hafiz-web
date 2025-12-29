---
title: Installation
description: Install Hafiz on your system
---

# Installation

## Docker (Recommended)

Since Hafiz is in active development, build the Docker image locally:

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
docker build -t hafiz:local .
```

Then run:

```bash
docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
  -e HAFIZ_ROOT_SECRET_KEY=hafizadmin \
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
curl https://hafiz.local:9000/health

# Access Admin UI
open https://hafiz.local:9000/admin
```
