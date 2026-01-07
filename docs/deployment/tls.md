---
title: TLS/HTTPS Configuration
description: Secure Hafiz with TLS certificates for production environments
---

# TLS/HTTPS Configuration

This guide covers configuring TLS/HTTPS for secure communication with Hafiz, including both client-to-server (S3 API) and inter-node cluster communication.

## Table of Contents

- [Overview](#overview)
- [Certificate Types](#certificate-types)
- [Quick Start: Self-Signed Certificates](#quick-start-self-signed-certificates)
- [Production: Let's Encrypt](#production-lets-encrypt)
- [Production: Custom CA](#production-custom-ca)
- [Configuration Options](#configuration-options)
- [Cluster TLS](#cluster-tls)
- [Client Configuration](#client-configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Overview

Hafiz supports TLS encryption for:

1. **S3 API Traffic** - Client applications connecting to the S3-compatible API
2. **Admin Panel** - Browser access to the web-based admin interface
3. **Cluster Communication** - Inter-node replication and health checks (multi-node deployments)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TLS Architecture                                   │
│                                                                              │
│   ┌──────────────┐                           ┌─────────────────────────────┐│
│   │   S3 Client  │──────HTTPS:9000──────────▶│                              ││
│   └──────────────┘                           │                              ││
│                                              │      Hafiz Server            ││
│   ┌──────────────┐                           │                              ││
│   │   Browser    │──────HTTPS:9000/admin────▶│   cert: server.crt          ││
│   │ (Admin Panel)│                           │   key:  server.key          ││
│   └──────────────┘                           │                              ││
│                                              └──────────────┬───────────────┘│
│                                                             │                │
│                                           Cluster TLS (mTLS)│                │
│                                                             │                │
│                                              ┌──────────────▼───────────────┐│
│                                              │      Node 2                   ││
│                                              │   cert: node2.crt            ││
│                                              │   ca:   ca.crt               ││
│                                              └───────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Certificate Types

| Type | Use Case | Pros | Cons |
|------|----------|------|------|
| **Self-Signed** | Development, Testing, Internal networks | Free, Quick setup | Browser warnings, Manual trust required |
| **Let's Encrypt** | Production with public domain | Free, Auto-renewal, Trusted by browsers | Requires public DNS, 90-day validity |
| **Custom CA** | Enterprise, Internal PKI | Full control, Long validity | Requires CA infrastructure |

---

## Quick Start: Self-Signed Certificates

### Single Node Setup

```bash
# Create certificates directory
mkdir -p /opt/hafiz/certs

# Generate private key and certificate (valid for 1 year)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/hafiz/certs/server.key \
  -out /opt/hafiz/certs/server.crt \
  -subj "/CN=hafiz.local/O=Hafiz" \
  -addext "subjectAltName=DNS:hafiz.local,DNS:localhost,IP:127.0.0.1"

# Set secure permissions
chmod 600 /opt/hafiz/certs/server.key
chmod 644 /opt/hafiz/certs/server.crt
```

### Enable TLS

**Option 1: Environment Variables**

```bash
HAFIZ_TLS_CERT=/opt/hafiz/certs/server.crt \
HAFIZ_TLS_KEY=/opt/hafiz/certs/server.key \
HAFIZ_PORT=9000 \
/opt/hafiz/hafiz-server
```

**Option 2: Configuration File**

Add to `hafiz.toml`:

```toml
[tls]
enabled = true
cert_file = "/opt/hafiz/certs/server.crt"
key_file = "/opt/hafiz/certs/server.key"
min_version = "1.2"
```

### Test HTTPS Connection

```bash
# Skip certificate verification (self-signed)
curl -k https://localhost:9000/health

# With CA certificate
curl --cacert /opt/hafiz/certs/server.crt https://localhost:9000/health
```

---

## Production: Let's Encrypt

### Prerequisites

- Domain name pointing to your server (e.g., `hafiz.example.com`)
- Port 80 accessible from the internet (for HTTP challenge)

### Obtain Certificate

```bash
# Install certbot
# Rocky Linux / RHEL
sudo dnf install -y certbot

# Ubuntu / Debian
sudo apt install -y certbot

# Obtain certificate (stop Hafiz first if using port 80)
sudo certbot certonly --standalone \
  -d hafiz.example.com \
  --agree-tos \
  --email admin@example.com
```

### Deploy Certificate

```bash
# Copy certificates (Let's Encrypt updates these files on renewal)
sudo cp /etc/letsencrypt/live/hafiz.example.com/fullchain.pem /opt/hafiz/certs/server.crt
sudo cp /etc/letsencrypt/live/hafiz.example.com/privkey.pem /opt/hafiz/certs/server.key

# Set permissions
sudo chmod 600 /opt/hafiz/certs/server.key
sudo chown hafiz:hafiz /opt/hafiz/certs/*
```

### Auto-Renewal Setup

```bash
# Create renewal hook
sudo tee /etc/letsencrypt/renewal-hooks/deploy/hafiz-cert.sh << 'EOF'
#!/bin/bash
DOMAIN="hafiz.example.com"
cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem /opt/hafiz/certs/server.crt
cp /etc/letsencrypt/live/$DOMAIN/privkey.pem /opt/hafiz/certs/server.key
chmod 600 /opt/hafiz/certs/server.key
chown hafiz:hafiz /opt/hafiz/certs/*
systemctl reload hafiz
EOF

chmod +x /etc/letsencrypt/renewal-hooks/deploy/hafiz-cert.sh

# Test renewal
sudo certbot renew --dry-run
```

---

## Production: Custom CA

For enterprise environments with internal PKI or multi-node clusters requiring mutual TLS.

### Step 1: Create Certificate Authority

```bash
mkdir -p /opt/hafiz/certs
cd /opt/hafiz/certs

# Generate CA private key
openssl genrsa -out ca.key 4096

# Generate CA certificate (valid for 10 years)
openssl req -x509 -new -nodes -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt \
  -subj "/C=US/ST=California/L=San Francisco/O=My Organization/OU=IT/CN=Hafiz CA"
```

### Step 2: Create Server Certificate

```bash
# Generate server private key
openssl genrsa -out server.key 2048

# Create Certificate Signing Request (CSR)
openssl req -new -key server.key \
  -out server.csr \
  -subj "/C=US/ST=California/L=San Francisco/O=My Organization/OU=Storage/CN=hafiz.example.com"

# Create extensions file for SANs
cat > server.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1 = hafiz.example.com
DNS.2 = localhost
IP.1 = 10.0.0.10
IP.2 = 127.0.0.1
EOF

# Sign the certificate with CA
openssl x509 -req -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt \
  -days 365 \
  -extfile server.ext

# Verify certificate
openssl x509 -in server.crt -text -noout

# Clean up
rm server.csr server.ext
```

### Step 3: Deploy Certificates

```bash
# Set permissions
chmod 600 /opt/hafiz/certs/*.key
chmod 644 /opt/hafiz/certs/*.crt
chown -R hafiz:hafiz /opt/hafiz/certs
```

---

## Configuration Options

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `HAFIZ_TLS_CERT` | Path to TLS certificate | `/opt/hafiz/certs/server.crt` |
| `HAFIZ_TLS_KEY` | Path to TLS private key | `/opt/hafiz/certs/server.key` |
| `HAFIZ_TLS_CA` | Path to CA certificate (for client verification) | `/opt/hafiz/certs/ca.crt` |

### Configuration File Options

```toml
[tls]
enabled = true
cert_file = "/opt/hafiz/certs/server.crt"
key_file = "/opt/hafiz/certs/server.key"
min_version = "1.2"           # Minimum TLS version: "1.2" or "1.3"
hsts_enabled = true           # HTTP Strict Transport Security
hsts_max_age = 31536000       # HSTS max-age in seconds (1 year)

# For mutual TLS (mTLS) - client certificate verification
client_ca_file = "/opt/hafiz/certs/ca.crt"
require_client_cert = false   # Set to true to require client certificates
```

### TLS Cipher Suites

Hafiz uses secure cipher suites by default. TLS 1.2+ is recommended for production.

---

## Cluster TLS

For multi-node deployments, enable TLS for inter-node communication.

### Generate Node Certificates

Each node needs its own certificate signed by the same CA.

```bash
# On each node, generate key and CSR
NODE_NAME="node1"
NODE_IP="10.50.0.10"
NODE_DNS="hafiz-node1.example.com"

openssl genrsa -out /opt/hafiz/certs/${NODE_NAME}.key 2048

# Create CSR
openssl req -new -key /opt/hafiz/certs/${NODE_NAME}.key \
  -out /opt/hafiz/certs/${NODE_NAME}.csr \
  -subj "/CN=${NODE_DNS}"

# Create extensions file
cat > /opt/hafiz/certs/${NODE_NAME}.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names

[alt_names]
DNS.1 = ${NODE_DNS}
DNS.2 = localhost
IP.1 = ${NODE_IP}
IP.2 = 127.0.0.1
EOF

# Sign with CA (run on CA server or copy files)
openssl x509 -req -in /opt/hafiz/certs/${NODE_NAME}.csr \
  -CA /opt/hafiz/certs/ca.crt -CAkey /opt/hafiz/certs/ca.key -CAcreateserial \
  -out /opt/hafiz/certs/${NODE_NAME}.crt \
  -days 365 \
  -extfile /opt/hafiz/certs/${NODE_NAME}.ext
```

### Configure Cluster TLS

**Node 1 Configuration (`hafiz.toml`):**

```toml
[tls]
enabled = true
cert_file = "/opt/hafiz/certs/node1.crt"
key_file = "/opt/hafiz/certs/node1.key"

[cluster]
enabled = true
name = "hafiz-cluster"
advertise_endpoint = "https://10.50.0.10:9000"
cluster_port = 9001
seed_nodes = []
cluster_tls_enabled = true
cluster_ca_cert = "/opt/hafiz/certs/ca.crt"
```

**Node 2 Configuration:**

```toml
[tls]
enabled = true
cert_file = "/opt/hafiz/certs/node2.crt"
key_file = "/opt/hafiz/certs/node2.key"

[cluster]
enabled = true
name = "hafiz-cluster"
advertise_endpoint = "https://10.50.0.11:9000"
cluster_port = 9001
seed_nodes = ["https://10.50.0.10:9000"]
cluster_tls_enabled = true
cluster_ca_cert = "/opt/hafiz/certs/ca.crt"
```

### Distribute CA Certificate

Copy `ca.crt` to all nodes:

```bash
# From CA server to all nodes
for node in node1 node2 node3; do
  scp /opt/hafiz/certs/ca.crt root@$node:/opt/hafiz/certs/
done
```

---

## Client Configuration

### AWS CLI with HTTPS

```bash
# With self-signed certificate (skip verification)
aws --endpoint-url https://localhost:9000 --no-verify-ssl s3 ls

# With CA certificate
aws --endpoint-url https://hafiz.example.com:9000 \
  --ca-bundle /opt/hafiz/certs/ca.crt \
  s3 ls
```

### Python (boto3) with HTTPS

```python
import boto3

# With custom CA
s3 = boto3.client(
    's3',
    endpoint_url='https://hafiz.example.com:9000',
    aws_access_key_id='hafizadmin',
    aws_secret_access_key='hafizadmin',
    verify='/opt/hafiz/certs/ca.crt'  # Path to CA cert
)

# Skip verification (development only!)
s3 = boto3.client(
    's3',
    endpoint_url='https://localhost:9000',
    aws_access_key_id='hafizadmin',
    aws_secret_access_key='hafizadmin',
    verify=False
)
```

### curl with HTTPS

```bash
# With CA certificate
curl --cacert /opt/hafiz/certs/ca.crt https://hafiz.example.com:9000/health

# Skip verification (development only)
curl -k https://localhost:9000/health
```

### rclone with HTTPS

```ini
[hafiz]
type = s3
provider = Other
endpoint = https://hafiz.example.com:9000
access_key_id = hafizadmin
secret_access_key = hafizadmin
# For self-signed certificates, add:
# no_check_certificate = true
```

---

## Verification

### Check Certificate Details

```bash
# View certificate information
openssl x509 -in /opt/hafiz/certs/server.crt -text -noout

# Check expiration date
openssl x509 -in /opt/hafiz/certs/server.crt -noout -enddate

# Verify certificate chain
openssl verify -CAfile /opt/hafiz/certs/ca.crt /opt/hafiz/certs/server.crt
```

### Test TLS Connection

```bash
# Test TLS handshake
openssl s_client -connect localhost:9000 -servername hafiz.example.com

# Check supported TLS versions
openssl s_client -connect localhost:9000 -tls1_2
openssl s_client -connect localhost:9000 -tls1_3
```

### Health Check via HTTPS

```bash
curl -k https://localhost:9000/health
# Expected: ok
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `certificate verify failed` | Self-signed cert not trusted | Use `--cacert` or `-k` flag |
| `certificate has expired` | Certificate past expiry date | Renew certificate |
| `hostname mismatch` | SAN doesn't include hostname/IP | Regenerate cert with correct SANs |
| `permission denied` | Key file permissions too open | Run `chmod 600 server.key` |
| `unable to load certificate` | Wrong file path or format | Verify path and PEM format |

### Certificate Debugging

```bash
# Check if certificate and key match
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
# Both outputs should match

# View certificate SANs
openssl x509 -in server.crt -noout -text | grep -A1 "Subject Alternative Name"

# Test specific TLS version
openssl s_client -connect localhost:9000 -tls1_3 2>&1 | head -20
```

### Logs

Enable debug logging to see TLS-related messages:

```bash
HAFIZ_LOG_LEVEL=debug HAFIZ_TLS_CERT=... HAFIZ_TLS_KEY=... hafiz-server
```

Look for:
- `TLS enabled with certificate:`
- `TLS handshake completed`
- `TLS error:`

---

## Security Best Practices

1. **Use TLS 1.2 or higher** - Disable older protocols
2. **Strong key sizes** - RSA 2048+ or ECDSA P-256+
3. **Secure key storage** - chmod 600, owned by service user
4. **Regular renewal** - Automate certificate renewal before expiry
5. **Monitor expiration** - Set up alerts for upcoming expirations
6. **HSTS headers** - Enable for browser security
7. **Certificate transparency** - Use trusted CAs for public deployments

---

## Next Steps

- [Single Node Deployment](./single-node.md) - Deploy with TLS on a single server
- [Cluster Deployment](./cluster.md) - Multi-node setup with TLS
- [Monitoring](../operations/monitoring.md) - Monitor TLS certificate expiration
