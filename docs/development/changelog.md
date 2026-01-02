---
title: Changelog
description: Version history
---

# Changelog

All notable changes to Hafiz will be documented here.

## [1.0.1] - 2026-01-02

### Enterprise Enhancements

#### IAM-Style User Bucket Access Policies
- Added `BucketPermission` enum with Read, Write, ReadWrite, None levels
- User-level bucket access control via Admin API
- Pattern-based bucket matching for wildcard permissions
- Service account support for application integrations

#### Unidirectional Replication
- Added `ReplicationDirection` for one-way sync (Primary → Replica)
- Support for read-only replica sites
- Disaster recovery with automatic failover procedures
- Replication service accounts with write-only access

#### Air-Gap Documentation
- Comprehensive air-gap deployment guide
- Export/import scripts for offline data transfer
- Checksum verification at every level
- Incremental sync support for ongoing transfers

#### Admin Panel Improvements
- Enhanced sidebar navigation with visible active state
- Browse Objects button on bucket cards
- User description field in user management
- Professional enterprise theme styling

#### Error Handling
- Comprehensive error code system with 50+ error types
- Internationalization support (EN/TR)
- S3-compatible error responses
- Detailed error messages for debugging

### Technical Details
- New types: `BucketPermission`, `BucketAccess`, `ReplicationDirection`
- API endpoint: `PUT /admin/users/:access_key/buckets`
- Full backwards compatibility maintained

[1.0.1]: https://github.com/shellnoq/hafiz/releases/tag/v1.0.1

---

## [1.0.0] - 2025-12-15

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
- Erasure coding for data durability
- Tiered storage (hot/warm/cold)
- Data deduplication
- Compression support

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
