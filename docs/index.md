---
title: Home
description: Hafiz - Enterprise-grade S3-compatible object storage written in Rust
---

# Hafiz

<p align="center">
  <img src="assets/logo.svg" alt="Hafiz Logo" width="200">
</p>

<p align="center">
  <strong>Enterprise-grade S3-compatible object storage written in Rust</strong>
</p>

<p align="center">
  <a href="https://github.com/shellnoq/hafiz/actions"><img src="https://github.com/shellnoq/hafiz/workflows/CI/badge.svg" alt="CI Status"></a>
  <a href="https://github.com/shellnoq/hafiz/releases"><img src="https://img.shields.io/github/v/release/shellnoq/hafiz" alt="Release"></a>
  <a href="https://github.com/shellnoq/hafiz/blob/main/LICENSE"><img src="https://img.shields.io/github/license/shellnoq/hafiz" alt="License"></a>
</p>

---

**Hafiz** (حافظ - "Guardian" in Arabic/Turkish) is a high-performance, S3-compatible object storage system. Built from the ground up in Rust for memory safety, performance, and reliability.

## Why Hafiz?

<div class="grid cards" markdown>

-   :material-lightning-bolt:{ .lg .middle } __High Performance__

    ---

    Built in Rust with async I/O for maximum throughput and minimal latency.

-   :material-shield-lock:{ .lg .middle } __Enterprise Security__

    ---

    AES-256-GCM encryption, Object Lock (WORM), LDAP integration.

-   :material-scale-balance:{ .lg .middle } __S3 Compatible__

    ---

    Works with AWS SDKs, CLI, and existing S3 tools.

-   :material-kubernetes:{ .lg .middle } __Cloud Native__

    ---

    Helm chart, Docker images, Prometheus metrics.

</div>

## Quick Start

=== "Docker (Build Locally)"

    ```bash
    # Clone and build
    git clone https://github.com/shellnoq/hafiz.git
    cd hafiz
    docker build -t hafiz:local .

    # Run
    docker run -d \
      --name hafiz \
      -p 9000:9000 \
      -v hafiz-data:/data \
      -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
      -e HAFIZ_ROOT_SECRET_KEY=hafizadmin \
      hafiz:local
    ```

    Access:

    - **S3 API**: http://localhost:9000
    - **Admin UI**: http://localhost:9000/admin
    - **File Browser**: http://localhost:9000/browse

=== "Docker Compose"

    ```bash
    git clone https://github.com/shellnoq/hafiz.git
    cd hafiz
    docker compose up -d
    ```

=== "Cluster Mode"

    ```bash
    git clone https://github.com/shellnoq/hafiz.git
    cd hafiz
    docker build -t hafiz:latest .
    docker compose -f docker-compose.cluster.yml up -d
    ```

=== "From Source"

    ```bash
    # Prerequisites: Rust 1.85+
    git clone https://github.com/shellnoq/hafiz.git
    cd hafiz
    cargo build --release

    HAFIZ_ROOT_ACCESS_KEY=admin \
    HAFIZ_ROOT_SECRET_KEY=password \
    ./target/release/hafiz-server
    ```

## Features

### Core Storage

| Feature | Status |
|---------|--------|
| Bucket Operations | :white_check_mark: |
| Object Operations | :white_check_mark: |
| Multipart Uploads | :white_check_mark: |
| Versioning | :white_check_mark: |
| Lifecycle Policies | :white_check_mark: |
| Storage Classes | :white_check_mark: |

### Security

| Feature | Status |
|---------|--------|
| AWS Signature V4 | :white_check_mark: |
| Server-Side Encryption | :white_check_mark: |
| Object Lock (WORM) | :white_check_mark: |
| Bucket Policies | :white_check_mark: |
| LDAP Integration | :white_check_mark: |
| TLS | :white_check_mark: |

### Operations

| Feature | Status |
|---------|--------|
| Admin API | :white_check_mark: |
| Web File Browser | :white_check_mark: |
| Presigned URLs | :white_check_mark: |
| Prometheus Metrics | :white_check_mark: |
| Event Notifications | :white_check_mark: |
| Access Logging | :white_check_mark: |
| Health Checks | :white_check_mark: |

### Enterprise

| Feature | Status |
|---------|--------|
| Multi-Server Cluster | :white_check_mark: |
| Cross-Network Replication | :white_check_mark: |
| Unidirectional Replication | :white_check_mark: |
| Air-Gapped Sync | :white_check_mark: |
| IAM-Style User Policies | :white_check_mark: |
| PostgreSQL Backend | :white_check_mark: |
| High Availability | :white_check_mark: |

## Comparison

| Feature | Hafiz | AWS S3 |
|---------|-------|--------|
| S3 Compatible | :white_check_mark: | :white_check_mark: |
| Object Lock | :white_check_mark: | :white_check_mark: |
| LDAP | :white_check_mark: | :x: |
| Written in Rust | :white_check_mark: | :x: |
| License | Apache 2.0 | N/A |
| Self-Hosted | :white_check_mark: | :x: |

## Integrations

Hafiz works with any S3-compatible application:

<div class="grid cards" markdown>

-   :material-cloud:{ .lg .middle } __Nextcloud__

    ---

    Use Hafiz as external storage for Nextcloud.

    [:octicons-arrow-right-24: Nextcloud Guide](integrations/nextcloud.md)

-   :material-backup-restore:{ .lg .middle } __Backup Tools__

    ---

    rclone, Veeam, Duplicati, and more.

    [:octicons-arrow-right-24: Integrations](integrations/index.md)

</div>

## Getting Help

- :fontawesome-brands-github: [GitHub Issues](https://github.com/shellnoq/hafiz/issues)
- :fontawesome-brands-github: [GitHub Discussions](https://github.com/shellnoq/hafiz/discussions)
- :material-file-document: [Documentation](getting-started/quickstart.md)

## License

Hafiz is open source under the [Apache License 2.0](https://github.com/shellnoq/hafiz/blob/main/LICENSE).
