---
title: Changelog
description: Version history
---

# Changelog

All notable changes to Hafiz will be documented here.

## [1.0.0] - 2025-01-15

### Production Ready Release

Hafiz is now production-ready for enterprise deployments.

### Core Features

- Full S3 API compatibility (76+ endpoints)
- Multi-part uploads with resumable support
- Object versioning with delete markers
- Lifecycle policies with automatic expiration
- Server-side encryption (AES-256-GCM)
- Object Lock (WORM) for compliance
- Bucket policies with IAM-style access control
- LDAP/Active Directory integration
- Admin API & Web UI
- Web File Browser
- Prometheus metrics & Grafana dashboards
- Event notifications (webhooks)
- Multi-server cluster mode
- Cross-network replication
- Air-gapped system support
- Helm chart for Kubernetes
- CLI tool (hafiz-cli)

### Technical Highlights

- 33,000+ lines of Rust
- 9 modular crates
- PostgreSQL & SQLite support
- Async I/O with Tokio
- Zero-copy streaming where possible
- Memory-safe by design

### Deployment Options

- Docker & Docker Compose
- Kubernetes with Helm
- Multi-server cluster
- Air-gapped environments

[1.0.0]: https://github.com/shellnoq/hafiz/releases/tag/v1.0.0
