# N8N + Evolution API + MinIO Template

A complete Docker-based integration template for building WhatsApp automation workflows using N8N, Evolution API, and MinIO object storage.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Service URLs](#service-urls)
- [Connecting Evolution API to N8N](#connecting-evolution-api-to-n8n)
- [Common Tasks](#common-tasks)
- [Troubleshooting](#troubleshooting)
- [Documentation Links](#documentation-links)

## Overview

This template provides a production-ready stack for WhatsApp automation:

- **N8N** - Workflow automation platform for creating integrations and automations
- **Evolution API** - WhatsApp Business API implementation for managing WhatsApp instances
- **MinIO** - S3-compatible object storage for files and media
- **PostgreSQL** - Database for Evolution API data persistence
- **Redis** - Caching layer for both N8N and Evolution API

### Use Cases

- Automated WhatsApp messaging systems
- Customer service chatbots
- Bulk messaging workflows
- WhatsApp-based notifications and alerts
- Multi-channel communication automation

## Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   Evolution API │◄────►│       N8N        │◄────►│     MinIO       │
│   (WhatsApp)    │      │   (Automation)   │      │  (Storage)      │
│   Port: 8080    │      │   Port: 5678     │      │  Port: 9000/1   │
└────────┬────────┘      └─────────┬────────┘      └─────────────────┘
         │                         │
         ▼                         ▼
┌─────────────────┐      ┌─────────────────┐
│   PostgreSQL    │      │      Redis      │
│   (Evolution)   │      │   (N8N Cache)   │
└─────────────────┘      └─────────────────┘
```

All services communicate via Docker internal network named `internal`.

## Prerequisites

- **Docker** (20.10+) and **Docker Compose** (2.0+)
  - [Install Docker](https://docs.docker.com/engine/install/)
  - [Install Docker Compose](https://docs.docker.com/compose/install/)
- **Ubuntu/Debian Linux** (or compatible)
- **Minimum 8GB RAM** (16GB recommended for high-volume workflows)
- **10GB free disk space**

## Quick Start

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd n8n-evolution-template
```

### 2. Configure Environment Variables

Copy the example environment files and customize them:

```bash
cp .env.example .env
cp .env.evolution.example .env.evolution
```

**Edit `.env`:**
```bash
nano .env
```

Update these critical values:
- `N8N_PASSWORD` - Your N8N admin password
- `MINIO_ROOT_USER` - MinIO admin username
- `MINIO_ROOT_PASSWORD` - MinIO admin password (min 8 characters)

**Edit `.env.evolution`:**
```bash
nano .env.evolution
```

Update this critical value:
- `AUTHENTICATION_API_KEY` - Strong API key for Evolution API (generate a random string)

### 3. Start the Services

```bash
docker compose up -d
```

### 4. Verify Services are Running

```bash
docker compose ps
```

You should see all 5 containers running:
- `n8n-core`
- `evolution-api-core`
- `minio-core`
- `evolution-postgres`
- `evolution-redis`
- `n8n-redis`

### 5. Access the Services

- **N8N:** http://localhost:5678
- **Evolution API:** http://localhost:8080
- **MinIO Console:** http://localhost:9001

## Configuration

### N8N Configuration

N8N is configured with:
- **Authentication:** Basic auth (username: `admin`, password: from `.env`)
- **Database:** SQLite (stored in Docker volume)
- **Payload Limit:** 4GB
- **Memory:** 64GB heap space for large workflows
- **Concurrent Workflows:** Up to 10 parallel executions

### Evolution API Configuration

Evolution API is configured with:
- **Database:** PostgreSQL (persistence for messages, contacts, chats)
- **Cache:** Redis (7-day TTL)
- **Webhook Events:** Messages, connections, QR codes
- **API Authentication:** Via `apikey` header

### MinIO Configuration

MinIO provides S3-compatible storage:
- **API Port:** 9000
- **Console Port:** 9001
- **Access:** Via credentials in `.env`

## Service URLs

### From Your Local Machine

- N8N: `http://localhost:5678`
- Evolution API: `http://localhost:8080`
- MinIO API: `http://localhost:9000`
- MinIO Console: `http://localhost:9001`

### From Inside Docker Containers

**Important:** Services must use Docker container names to communicate:

- N8N from Evolution: `http://n8n-core:5678`
- Evolution from N8N: `http://evolution-api-core:8080`
- MinIO from N8N: `http://minio-core:9000`

## Connecting Evolution API to N8N

### Step 1: Create a Webhook Workflow in N8N

1. Access N8N at http://localhost:5678
2. Create a new workflow
3. Add a **Webhook** node
4. Configure the webhook:
   - **HTTP Method:** POST
   - **Path:** (auto-generated or custom)
5. **ACTIVATE** the workflow (toggle in top-right corner)
6. Copy the **Production URL** (not test URL!)

Example Production URL:
```
http://localhost:5678/webhook/651c501f-a6bf-4a10-be2b-7af13cf37d39
```

### Step 2: Create a WhatsApp Instance

Use the Evolution API to create a WhatsApp instance:

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY_FROM_ENV" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-whatsapp-bot",
    "token": "optional-token",
    "qrcode": true
  }'
```

Response will include a QR code - scan it with WhatsApp to connect.

### Step 3: Configure Webhook for the Instance

**IMPORTANT:** Use `http://n8n-core:5678` (not localhost) since Evolution runs in Docker:

```bash
curl -X POST http://localhost:8080/webhook/set/my-whatsapp-bot \
  -H "apikey: YOUR_API_KEY_FROM_ENV" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://n8n-core:5678/webhook/YOUR-WEBHOOK-ID",
    "webhook_by_events": false,
    "events": [
      "MESSAGES_UPSERT",
      "MESSAGES_UPDATE",
      "CONNECTION_UPDATE"
    ]
  }'
```

### Step 4: Test the Integration

Send a WhatsApp message to your connected number and check the N8N workflow execution!

## Common Tasks

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f n8n-core
docker compose logs -f evolution-api-core
```

### Restart Services

```bash
# All services
docker compose restart

# Specific service
docker compose restart evolution-api-core
```

### Stop Services

```bash
docker compose down
```

### Update Services

```bash
docker compose pull
docker compose up -d
```

### Backup Data

```bash
# Backup all volumes
docker run --rm \
  -v n8n-evolution-template_n8n-data:/data/n8n \
  -v n8n-evolution-template_evolution-postgres:/data/postgres \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/backup-$(date +%Y%m%d).tar.gz /data
```

### List Evolution Instances

```bash
curl -X GET http://localhost:8080/instance/fetchInstances \
  -H "apikey: YOUR_API_KEY_FROM_ENV"
```

### Send a WhatsApp Message via Evolution

```bash
curl -X POST http://localhost:8080/message/sendText/my-whatsapp-bot \
  -H "apikey: YOUR_API_KEY_FROM_ENV" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "text": "Hello from Evolution API!"
  }'
```

## Troubleshooting

### Evolution Not Sending Webhooks to N8N

**Problem:** Evolution receives messages but N8N doesn't trigger

**Solution:**
1. Check N8N workflow is **ACTIVATED** (not just saved)
2. Verify you're using the **Production URL** (not test URL)
3. Ensure webhook URL uses `http://n8n-core:5678` (not localhost)
4. Check N8N logs: `docker compose logs n8n-core | grep webhook`

### N8N Shows "Webhook Not Registered"

**Problem:** 404 errors in N8N logs

**Causes:**
- Workflow is not activated
- Webhook path is incorrect
- Workflow was deleted but Evolution still has old URL

**Solution:**
1. Activate the workflow in N8N
2. Update Evolution webhook configuration with correct URL

### Evolution Instance Disconnected

**Problem:** WhatsApp instance shows "disconnected" or "401 Unauthorized"

**Causes:**
- WhatsApp removed device (logged out from phone)
- QR code session expired

**Solution:**
1. Delete the instance:
   ```bash
   curl -X DELETE http://localhost:8080/instance/delete/my-whatsapp-bot \
     -H "apikey: YOUR_API_KEY"
   ```
2. Create a new instance and scan QR code again

### Docker Network Issues

**Problem:** Services can't communicate

**Solution:**
```bash
# Check network exists
docker network ls | grep internal

# Inspect network
docker network inspect n8n-evolution-template_internal

# Restart services
docker compose down && docker compose up -d
```

### High Memory Usage

**Problem:** N8N consuming too much RAM

**Solution:**
1. Reduce `NODE_OPTIONS` heap size in docker-compose.yml
2. Decrease `N8N_PAYLOAD_SIZE_MAX` for smaller payloads
3. Limit concurrent workflows with `N8N_CONCURRENCY_PRODUCTION_LIMIT`

## Documentation Links

### Official Documentation

- **N8N Documentation:** https://docs.n8n.io/
  - [Self-hosting Guide](https://docs.n8n.io/hosting/)
  - [Webhook Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
  - [HTTP Request Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)
  - [Environment Variables](https://docs.n8n.io/hosting/configuration/environment-variables/)

- **Evolution API Documentation:** https://doc.evolution-api.com/
  - [API Reference](https://doc.evolution-api.com/v2/pt/api)
  - [Webhooks](https://doc.evolution-api.com/v2/pt/webhooks)
  - [Instance Management](https://doc.evolution-api.com/v2/pt/instance)
  - [Sending Messages](https://doc.evolution-api.com/v2/pt/messages/send-message-text)

- **MinIO Documentation:** https://min.io/docs/minio/linux/index.html
  - [Docker Deployment](https://min.io/docs/minio/container/index.html)
  - [S3 API Compatibility](https://min.io/docs/minio/linux/developers/python/API.html)

### Docker Resources

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Networking](https://docs.docker.com/network/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)

### Community Resources

- [N8N Community Forum](https://community.n8n.io/)
- [N8N GitHub](https://github.com/n8n-io/n8n)
- [Evolution API GitHub](https://github.com/EvolutionAPI/evolution-api)

## Environment Variables Reference

### `.env` - N8N & MinIO

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `N8N_PASSWORD` | N8N admin password | `changeme` | Yes |
| `N8N_SECURE_COOKIE` | Use secure cookies (true for HTTPS) | `false` | No |
| `MINIO_ROOT_USER` | MinIO admin username | `your-minio-user` | Yes |
| `MINIO_ROOT_PASSWORD` | MinIO admin password (min 8 chars) | `your-minio-password` | Yes |

### `.env.evolution` - Evolution API

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `AUTHENTICATION_API_KEY` | API key for Evolution API | `changeme` | Yes |
| `SERVER_PORT` | Evolution API port | `8080` | No |
| `DATABASE_PROVIDER` | Database type | `postgresql` | No |
| `WEBHOOK_GLOBAL_ENABLED` | Enable global webhooks | `false` | No |
| `CACHE_REDIS_ENABLED` | Enable Redis cache | `true` | No |

See `.env.example` and `.env.evolution.example` for complete variable lists.

## License

This template is provided as-is for use in your projects.

## Contributing

Feel free to submit issues or pull requests for improvements!

## Support

For issues specific to:
- **N8N:** Check [N8N Community Forum](https://community.n8n.io/)
- **Evolution API:** Check [Evolution API Docs](https://doc.evolution-api.com/)
- **This Template:** Open an issue in this repository
