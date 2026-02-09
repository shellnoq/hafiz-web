---
title: Components
description: Hafiz crate structure
---

# Components

## Crate Structure

```
hafiz/
├── crates/
│   ├── hafiz-core/       # Core types
│   ├── hafiz-s3-api/     # S3 API
│   ├── hafiz-storage/    # Storage backends
│   ├── hafiz-metadata/   # Database layer
│   ├── hafiz-auth/       # Authentication
│   ├── hafiz-crypto/     # Encryption
│   ├── hafiz-cluster/    # Clustering
│   ├── hafiz-admin/      # Admin API
│   └── hafiz-cli/        # CLI tool
```

## hafiz-core

Foundation crate with shared types:

- `Bucket`, `Object`, `ObjectMetadata`
- `HafizError` - Error types
- `Config` - Configuration

## hafiz-s3-api

HTTP layer using Axum:

- 76+ S3 API endpoints
- XML parsing
- Middleware

## hafiz-storage

Storage abstraction:

- `FilesystemStorage`
- `S3ProxyStorage`

## hafiz-metadata

Database layer using SQLx:

- `SqliteRepository`
- `PostgresRepository`

## hafiz-auth

Authentication:

- AWS Signature V4
- LDAP integration
- Policy evaluation

## hafiz-crypto

Encryption:

- AES-256-GCM
- Key derivation
- Hashing

## hafiz-cluster

Cluster coordination:

- Gossip-based node discovery
- Native object replication
- Erasure coding (Reed-Solomon)
- Shard distribution

## hafiz-transform

Object transform pipeline:

- Thumbnail generation (image crate)
- EXIF metadata stripping
- Image format conversion
- Async pipeline execution
