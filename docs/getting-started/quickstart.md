---
title: Quick Start
description: Get Hafiz running in 5 minutes
---

# Quick Start

Get Hafiz running in under 5 minutes.

## Prerequisites

- Docker (recommended) or
- Rust 1.85+ (for building from source)

## Docker (Build Locally)

Since Hafiz is still in active development, you'll need to build the Docker image locally:

```bash
# Clone and build
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
docker build -t hafiz:local .

# Run container with persistent storage
docker run -d \
  --name hafiz \
  -p 9000:9000 \
  -v hafiz-data:/data \
  -e HAFIZ_ROOT_ACCESS_KEY=hafizadmin \
  -e HAFIZ_ROOT_SECRET_KEY=hafizadmin \
  hafiz:local
```

That's it! Hafiz is now running:

- **S3 API**: http://localhost:9000
- **Admin UI**: http://localhost:9000/admin
- **Metrics**: http://localhost:9000/metrics

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

## Docker Compose

For a quick start with Docker Compose:

```bash
git clone https://github.com/shellnoq/hafiz.git
cd hafiz
docker compose up -d
```

For a production setup with PostgreSQL, use the cluster configuration:

```bash
# Build image first
docker build -t hafiz:latest .

# Run cluster with PostgreSQL and HAProxy
docker compose -f docker-compose.cluster.yml up -d
```

Access via load balancer:

- **S3 API**: http://localhost (via HAProxy)
- **HAProxy Stats**: http://localhost:8404

## Next Steps

- [Configuration](configuration.md) - Customize your deployment
- [User Guide](../user-guide/index.md) - Learn about buckets and objects
- [Encryption](../user-guide/encryption.md) - Enable server-side encryption
- [Access Control](../user-guide/access-control.md) - Configure permissions
