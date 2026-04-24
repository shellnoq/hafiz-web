---
title: Quick Start
description: Get Hafiz running in 5 minutes
---

# Quick Start

Hafiz up and running in three commands.

## Prerequisites

- Docker Engine (compose v2) — only real requirement
- `openssl` if you're deploying the cluster (generates the peer-auth secret)

## Single-node

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
cp .env.example .env                    # edit: set HAFIZ_ROOT_SECRET_KEY
docker compose up -d
```

Hafiz is now running:

- **S3 API**: http://localhost:9000
- **Admin UI**: http://localhost:9000/admin
- **Metrics**: http://localhost:9000/metrics

## Cluster (3 nodes + PostgreSQL + HAProxy)

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
cp .env.example .env
echo "HAFIZ_CLUSTER_SHARED_SECRET=$(openssl rand -hex 32)" >> .env
docker compose -f docker-compose.cluster.yml up -d
```

The cluster file requires the shared secret — compose exits with a clear error if it's missing, so you can't accidentally ship an unauthenticated deployment. See [Cluster Peer Auth](../deployment/cluster-auth.md) for details.

- **S3 API (via HAProxy)**: http://localhost
- **HAProxy Stats**: http://localhost:8404
- **Direct node access**: 9000, 9010, 9020

## Verify Installation

### Using AWS CLI

```bash
# Configure
aws configure set aws_access_key_id hafizadmin
aws configure set aws_secret_access_key hafizadmin

# Create bucket
aws --endpoint-url http://localhost:9000 s3 mb s3://my-bucket

# Upload file
echo "Hello, Hafiz!" > test.txt
aws --endpoint-url http://localhost:9000 s3 cp test.txt s3://my-bucket/

# List objects
aws --endpoint-url http://localhost:9000 s3 ls s3://my-bucket/
```

### Using Python

```python
import boto3

s3 = boto3.client(
    's3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='hafizadmin',
    aws_secret_access_key='hafizadmin'
)

# Create bucket
s3.create_bucket(Bucket='my-bucket')

# Upload
s3.put_object(Bucket='my-bucket', Key='hello.txt', Body=b'Hello!')

# Download
response = s3.get_object(Bucket='my-bucket', Key='hello.txt')
print(response['Body'].read())
```

### Using Hafiz CLI

```bash
# Install
cargo install hafiz-cli

# Configure
hafiz configure
# Endpoint: http://localhost:9000
# Access Key: hafizadmin
# Secret Key: hafizadmin

# Use
hafiz ls s3://
hafiz mb s3://my-bucket
hafiz cp file.txt s3://my-bucket/
```

## Next Steps

- [Configuration](configuration.md) - Customize your deployment
- [User Guide](../user-guide/index.md) - Learn about buckets and objects
- [Encryption](../user-guide/encryption.md) - Enable server-side encryption
- [Access Control](../user-guide/access-control.md) - Configure permissions
