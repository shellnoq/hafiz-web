---
title: Roadmap
description: Hafiz development roadmap
---

# Roadmap

## Current Status: v0.3.0

Hafiz v0.3.0 includes 9 major new features including S3 Select, compression, deduplication, audit logging, and erasure coding.

---

## v0.1.0 (Released Dec 2024)

- [x] Core S3 API (76+ endpoints)
- [x] Versioning
- [x] Server-side Encryption (AES-256-GCM)
- [x] Object Lock (WORM)
- [x] Helm Chart
- [x] CLI Tool
- [ ] CI/CD Pipeline
- [ ] Integration Tests
- [ ] Performance Benchmarks
- [ ] Docker Image Optimization
- [ ] ARM64 Support

## v0.2.0 (Released Feb 2026 - Replication & Cluster)

- [x] Native async object replication across cluster nodes
- [x] PostgreSQL backend for cluster operations
- [x] 3-node distributed deployment with HAProxy
- [x] 52 S3 API tests + 29 replication tests (all passing)

## v0.3.0 (Released Feb 2026 - Advanced Features)

- [x] S3 Select (SQL queries on CSV/JSON objects)
- [x] Object-Level Compression (zstd with smart content-type detection)
- [x] Data Deduplication (SHA-256 content-addressable storage)
- [x] Immutable Audit Log (blockchain-style hash chain)
- [x] Real-Time Change Stream (Server-Sent Events)
- [x] Temporary Bucket Sharing (token-based public access)
- [x] Object Transform Pipeline (thumbnail, EXIF strip, format conversion)
- [x] Erasure Coding (Reed-Solomon data protection)
- [x] PostgreSQL Error Logging (complete implementation)

## v1.0.0 (Q2 2026 - Production Ready)

### Goals
- Production stability
- Comprehensive documentation
- Enterprise support readiness

### Requirements
- [ ] 99.99% API compatibility
- [ ] Performance benchmarks published
- [ ] Security audit completed
- [ ] Disaster recovery tested
- [ ] Migration tools available

### Long-Term Support
- 2-year LTS commitment
- Security patches
- Critical bug fixes

---

## Future Considerations (v2.0+)

### Potential Features
- **S3 Express One Zone** - Ultra-low latency storage class
- **Intelligent Tiering** - ML-based data placement
- **Global Namespace** - Unified view across regions
- **Object Federation** - Query across buckets
- **Event Bridge** - Advanced event routing
- **Data Catalog** - Automatic schema discovery

### Performance Goals
- 10,000+ requests/second per node
- Sub-millisecond metadata operations
- 10 Gbps+ throughput per node

### Scalability Goals
- 1000+ node clusters
- Exabyte-scale storage
- Billions of objects

---

## Release Schedule

| Version | Target Date | Status |
|---------|-------------|--------|
| v0.1.0 | Dec 2024 | Released |
| v0.2.0 | Feb 2026 | Released |
| v0.3.0 | Feb 2026 | Released |
| v1.0.0 | Q2 2026 | Planned |

---

## Feature Requests

Open a [Discussion](https://github.com/shellnoq/hafiz/discussions) to propose new features.

### Priority Areas

1. **Testing** - Integration tests, load tests
2. **Documentation** - Examples, tutorials
3. **Performance** - Profiling, optimization
4. **Compatibility** - S3 API edge cases
