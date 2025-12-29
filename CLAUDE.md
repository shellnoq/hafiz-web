# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the documentation and landing page website for **Hafiz** - an enterprise-grade S3-compatible object storage system written in Rust. The main Hafiz source code lives at https://github.com/shellnoq/hafiz.

This repository contains:
- `index.html` - Static landing page with product marketing and quick start info
- `docs/` - Markdown documentation (MkDocs Material format)
- Hosted on GitHub Pages at `docs.hafiz.e2esolutions.tech`

## Repository Structure

```
hafiz-web/
├── index.html       # Landing page (vanilla HTML/CSS/JS)
├── docs/            # Documentation in Markdown
│   ├── api/         # S3 API reference
│   ├── architecture/# System architecture
│   ├── cli/         # CLI commands
│   ├── deployment/  # Docker, Kubernetes, cluster setup
│   ├── development/ # Contributing guidelines
│   ├── getting-started/
│   ├── operations/  # Monitoring, backup, troubleshooting
│   └── user-guide/  # Buckets, objects, encryption, versioning
├── CNAME            # Custom domain config
└── .nojekyll        # Disables Jekyll on GitHub Pages
```

## Key Context

The documentation describes Hafiz's Rust codebase which uses:
- **Build**: `cargo build` / `cargo build --release`
- **Test**: `cargo test`
- **Lint**: `cargo clippy`
- **Format**: `cargo fmt`
- **Run**: `cargo run --bin hafiz-server`

Hafiz architecture (multi-crate workspace):
- `hafiz-core` - Shared types (Bucket, Object, Config, errors)
- `hafiz-s3-api` - Axum-based S3 API (76+ endpoints)
- `hafiz-storage` - Storage backends (Filesystem, S3Proxy)
- `hafiz-metadata` - SQLx database layer (SQLite, PostgreSQL)
- `hafiz-auth` - AWS Signature V4, LDAP, policy evaluation
- `hafiz-crypto` - AES-256-GCM encryption
- `hafiz-cluster` - Distributed clustering
- `hafiz-admin` - Admin API
- `hafiz-cli` - CLI tool

## Documentation Style

- Uses MkDocs Material syntax with admonitions and tabbed code blocks
- Frontmatter with `title` and `description` fields
- Conventional commits: `feat:`, `fix:`, `docs:`, `test:`
