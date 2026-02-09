---
title: Changelog
description: Version history
---

# Changelog

## [0.3.0] - 2026-02-09

### Added

- **S3 Select (SQL on CSV/JSON Objects)**
  - `POST /{bucket}/{key}?select&select-type=2` endpoint for SQL queries
  - CSV input with configurable delimiter, quote character, and header detection
  - JSON input in DOCUMENT and LINES modes
  - SQL subset: SELECT, WHERE (=, !=, <, >, LIKE, IS NULL, AND, OR), LIMIT

- **Object-Level Compression (zstd)**
  - Automatic zstd compression on PUT with configurable level (1-22)
  - Transparent decompression on GET (including range requests)
  - Smart skip for already-compressed content types (images, video, zip)
  - Config: `HAFIZ_COMPRESSION_ENABLED`, `HAFIZ_COMPRESSION_LEVEL`

- **Data Deduplication (SHA-256)**
  - Content-addressable storage with SHA-256 hash-based dedup
  - Reference counting for shared storage of identical content
  - Config: `HAFIZ_DEDUP_ENABLED`, configurable min_size

- **Immutable Audit Log (Blockchain-Style)**
  - Every mutating operation logged with blockchain-style hash chain
  - Tamper detection via chain verification endpoint
  - Admin API: `GET /api/v1/audit/logs` and `GET /api/v1/audit/verify`

- **Real-Time Change Stream (SSE)**
  - Server-Sent Events: `GET /events/sse?bucket=optional-bucket`
  - Global and per-bucket event channels via tokio broadcast
  - Events for object creation and deletion with full metadata

- **Temporary Bucket Sharing**
  - Token-based public bucket access without S3 credentials
  - Admin API: `POST/GET/DELETE /api/v1/shares`
  - Public endpoints: `GET/PUT /share/:token/*key` (no auth required)
  - Configurable permissions, prefix restrictions, expiration, download limits

- **Object Transform Pipeline**
  - New `hafiz-transform` crate for async image processing
  - Built-in transforms: Thumbnail, EXIF strip, Format conversion
  - Per-bucket transform configuration via Admin API
  - Supported formats: JPEG, PNG, WebP, GIF, BMP

- **Erasure Coding (Reed-Solomon)**
  - Reed-Solomon encoding/decoding in `hafiz-cluster` crate
  - Configurable data shards (default 4) and parity shards (default 2)
  - Shard distribution across cluster nodes (round-robin placement)

- **PostgreSQL Error Logging**
  - Full implementation of 9 previously stubbed error logging methods
  - `error_logs` table with indexes for efficient querying
  - Aggregation, maintenance, and query-by-request-id support

### Technical

- 40,000+ lines of Rust
- 10 crates (new: hafiz-transform)
- 90+ API endpoints
- 30+ database tables
- New dependencies: reed-solomon-erasure, image, csv, tokio-stream, parking_lot

[0.3.0]: https://github.com/shellnoq/hafiz/releases/tag/v0.3.0

## [0.2.0] - 2026-02-07

### Added

- **Native Object Replication**
  - Async object replication across cluster nodes via `hafiz-cluster` crate
  - Automatic replication of PutObject, DeleteObject, CompleteMultipartUpload
  - Default "replicate everything" behavior with loop prevention
  - Cluster internal HTTP routes (`/cluster/join`, `/cluster/heartbeat`, etc.)

- **PostgreSQL Backend - Cluster Support**
  - `cluster_nodes` and `replication_rules` tables
  - Full CRUD operations for node registry and replication rules
  - PostgreSQL stubs for CORS, ACL, notification, policy, object lock

- **Distributed Deployment**
  - 3-node cluster deployment configs (`deploy/distributed/`)
  - HAProxy load balancer with consistent hashing
  - PostgreSQL initialization script

### Fixed

- Multipart upload replication and primary key mismatch
- User metadata persistence and HeadObject metadata headers
- Shared PostgreSQL metadata overwrite for REPLICA requests
- HAProxy balancing mode (roundrobin to source with consistent hashing)
- Bucket tagging dispatcher in routes

### Test Results

- 52/52 S3 API tests passing
- 29/29 Replication tests passing

### Technical

- 36,000+ lines of Rust
- Enhanced cluster replication protocol

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
