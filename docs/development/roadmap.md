---
title: Roadmap
description: Hafiz development roadmap
---

# Roadmap

## v1.0.1 - Enterprise Enhancements :white_check_mark:

Released: January 2026

### IAM-Style Access Control

- [x] User-level bucket access policies (Read/Write/ReadWrite/None)
- [x] Pattern-based bucket matching with wildcards
- [x] Service account support for applications
- [x] User description and metadata fields

### Replication Enhancements

- [x] Unidirectional replication (Primary → Replica)
- [x] Read-only replica site configuration
- [x] Replication direction modes (Bidirectional, SourceToDestination, DestinationToSource)
- [x] Automatic failover procedures

### Documentation & Tooling

- [x] Comprehensive air-gap deployment guide
- [x] Export/import scripts for offline transfers
- [x] Incremental sync for air-gapped environments
- [x] Enhanced cluster node addition guide

### Admin Panel

- [x] Professional theme with visible navigation state
- [x] Browse Objects button on bucket cards
- [x] User description visibility in UI
- [x] Improved active tab indicators

### Error Handling

- [x] Comprehensive error code system (50+ error types)
- [x] Internationalization support (English/Turkish)
- [x] S3-compatible error responses
- [x] Detailed debugging messages

---

## v1.0.0 - Production Ready :white_check_mark:

Released: December 2025

Hafiz v1.0.0 is **production-ready** and fully supported for enterprise deployments.

### Core Platform

- [x] Full S3 API compatibility (76+ endpoints)
- [x] Object versioning & delete markers
- [x] Lifecycle policies
- [x] Server-side encryption (AES-256-GCM)
- [x] Object Lock (WORM compliance)
- [x] Bucket policies & IAM-style access control
- [x] Multi-part uploads
- [x] Presigned URLs
- [x] Erasure coding
- [x] Tiered storage (hot/warm/cold)
- [x] Data deduplication
- [x] Compression

### Enterprise Features

- [x] LDAP/Active Directory integration
- [x] Multi-server cluster mode
- [x] Cross-network replication
- [x] Air-gapped system support
- [x] PostgreSQL backend for high availability
- [x] Prometheus metrics & monitoring
- [x] Admin API & Web UI
- [x] Web File Browser

### Deployment

- [x] Docker & Docker Compose
- [x] Kubernetes with Helm chart
- [x] Rocky Linux / RHEL deployment scripts
- [x] HAProxy load balancing
- [x] TLS/SSL support

---

## v1.1.0 - Extended Integrations (Q2 2026)

- [ ] Terraform provider
- [ ] Ansible roles
- [ ] OpenTelemetry tracing

## v1.2.0 - Advanced Features (Q4 2026)

- [ ] S3 Select (query objects with SQL)
- [ ] Vault integration for key management

---

## Enterprise Support

For commercial deployments, we offer:

- Priority support & SLAs
- Custom feature development
- On-premise deployment assistance
- Training & consulting

[Contact us](mailto:info@e2esolutions.tech) for enterprise inquiries.

## Feature Requests

Have a feature idea? Open a [Discussion](https://github.com/shellnoq/hafiz/discussions) to share your suggestions.
