---
title: Changelog
description: Version history
---

# Changelog

## [0.2.0] - 2025-01-01

### Added

- **Cluster Management**
  - Node drain API for graceful maintenance
  - Node removal from cluster
  - Real-time system stats collection (CPU, memory, disk)
  - Cluster stats computation from metadata

- **IAM Policy Conditions**
  - Full condition evaluation in bucket policies
  - String operators: StringEquals, StringNotEquals, StringLike, etc.
  - Numeric operators: NumericEquals, NumericLessThan, etc.
  - IP Address operators with CIDR notation support
  - Date operators: DateEquals, DateLessThan, etc.
  - Bool and Null condition operators

- **Cluster Transport**
  - Delete API for object replication
  - Custom CA certificate support for TLS
  - Client certificate authentication

### Improved

- Admin stats now show server-level encryption status
- Cluster heartbeat includes real node statistics
- Replication delete operations with progress tracking

### Technical

- 36,000+ lines of Rust
- Enhanced cluster replication protocol
- Cross-platform stats collection (Linux optimized)

[0.2.0]: https://github.com/shellnoq/hafiz/releases/tag/v0.2.0

## [0.1.0] - 2024-12-08

### Added

- Full S3 API compatibility (76+ endpoints)
- Multi-part uploads
- Object versioning
- Lifecycle policies
- Server-side encryption (AES-256-GCM)
- Object Lock (WORM)
- Bucket policies
- LDAP integration
- Admin API & UI
- Prometheus metrics
- Event notifications
- Cluster mode
- Helm chart
- CLI tool

### Technical

- 33,000+ lines of Rust
- 9 crates
- PostgreSQL & SQLite support

[0.1.0]: https://github.com/shellnoq/hafiz/releases/tag/v0.1.0
