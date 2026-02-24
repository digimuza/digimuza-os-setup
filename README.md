# digimuza-os-setup

A reusable, open-source AI infrastructure blueprint deployed via [Coolify](https://coolify.io). This repository contains Docker Compose configurations for 9 self-hosted services that together form a complete operational stack spanning authentication, AI chat, CRM, project management, automation, observability, and more.

Each service is packaged as a standalone `compose.yml` with its own `.env` file, designed to be deployed as an individual Coolify resource while sharing common infrastructure through Coolify's project-level environment variables.

## Services

| Service | Directory | Subdomain | Image | Purpose |
|---------|-----------|-----------|-------|---------|
| **Authentik** | `config/auth/` | `auth.DOMAIN` | `ghcr.io/goauthentik/server` | SSO & Identity Provider |
| **LibreChat** | `config/librechat/` | `chat.DOMAIN` | `ghcr.io/danny-avila/librechat-dev-api` | AI Chat Interface (multi-model) |
| **Twenty CRM** | `config/twentycrm/` | `twenty.DOMAIN` | `twentycrm/twenty` | CRM |
| **MinIO** | `config/minio/` | `minio.DOMAIN` | `ghcr.io/coollabsio/minio` | S3-compatible Object Storage |
| **Affine** | `config/affine/` | `affine.DOMAIN` | `ghcr.io/toeverything/affine` | Docs & Whiteboards |
| **N8N** | `config/n8n/` | `n8n.DOMAIN` | `n8nio/n8n` | Workflow Automation |
| **Trigger.dev** | `config/trigger.dev/` | `trigger.DOMAIN` | `ghcr.io/triggerdotdev/trigger.dev` | Background Jobs & Scheduling |
| **Plane** | `config/plane/` | `plane.DOMAIN` | `artifacts.plane.so/makeplane/plane-*` | Project Management |
| **SigNoz** | `config/signoz/` | `signoz.DOMAIN` | `signoz/signoz` | Observability & Monitoring |

## Architecture

All services are deployed as separate Coolify resources on a single host. They share:

- **Coolify external network** -- Services that need to communicate across compose stacks use the `coolify` external Docker network.
- **PostgreSQL** -- Most services run their own Postgres 16 instance (Alpine). Affine uses `pgvector/pgvector:pg16`; Plane uses Postgres 15.
- **Redis / Valkey** -- Used by N8N, Twenty CRM, Affine, Trigger.dev, and Plane for caching and queuing. Plane uses Valkey as a Redis-compatible alternative.
- **MinIO** -- Centrally deployed in `config/minio/` and consumed by other services (Twenty CRM, Plane) for S3-compatible object storage.
- **Cloudflare AI Gateway** -- Configured in `project.env` for proxying LLM API calls.
- **Resend** -- Shared SMTP/email provider configured in `project.env` and consumed by services that send email (Authentik, Trigger.dev, etc.).
- **ClickHouse** -- Used by both Trigger.dev and SigNoz for analytics and telemetry storage.

### Service-specific extras

- **LibreChat** includes MongoDB, Meilisearch, pgvector (RAG), and a RAG API sidecar.
- **Trigger.dev** includes ElectricSQL, ClickHouse, a supervisor, and a Docker socket proxy for managed worker execution.
- **N8N** runs with a separate worker and external task runner containers.
- **Plane** runs proxy, web, space, admin, live, API, worker, beat-worker, migrator, its own MinIO, and RabbitMQ.
- **SigNoz** runs with Zookeeper, ClickHouse, an OpenTelemetry collector, and schema migrators.

## Repository structure

```
config/
  project.env              # Shared variables (Coolify project-level env)
  auth/
    compose.yml            # Authentik server + worker + Postgres
    .env                   # Service-specific env vars
  librechat/
    compose.yml            # LibreChat + MongoDB + Meilisearch + pgvector + RAG API
    .env
  twentycrm/
    compose.yml            # Twenty CRM + worker + Postgres + Redis
    .env
  minio/
    compose.yml            # MinIO object storage
    .env
  affine/
    compose.yml            # Affine + migration + pgvector + Redis
    .env
  n8n/
    compose.yml            # N8N + worker + task-runners + Postgres + Redis
    .env
  trigger.dev/
    compose.yml            # Trigger.dev + supervisor + Electric + ClickHouse + Postgres + Redis
    .env
  plane/
    compose.yml            # Plane (proxy, web, space, admin, live, API, workers) + Postgres + Valkey + RabbitMQ + MinIO
    .env
  signoz/
    compose.yml            # SigNoz + OTel Collector + ClickHouse + Zookeeper + schema migrators
    .env
```

## Configuration

### Project-level variables (`config/project.env`)

Shared configuration that applies across all services is defined in `config/project.env`. In Coolify, these are set as **project-level environment variables** and referenced in per-service `.env` files using Coolify's `{{project.VARIABLE}}` templating syntax.

Key shared variables include:

| Variable | Purpose |
|----------|---------|
| `DOMAIN` | Base domain for all subdomains |
| `AI_GATEWAY` | Cloudflare AI Gateway URL |
| `RESEND_API_KEY` | Shared email provider API key |
| `RESEND_SMTP_HOST` / `RESEND_SMTP_PORT` | SMTP configuration |
| `FROM_EMAIL` / `REPLY_TO_EMAIL` | Default sender addresses |
| `STORAGE_S3_ENDPOINT` | MinIO S3 API endpoint |
| `STORAGE_S3_ACCESS_KEY_ID` / `STORAGE_S3_SECRET_ACCESS_KEY` | MinIO credentials |
| `OAUTH_CLIENT_ID` / `OAUTH_SECRET` | Authentik OAuth credentials for SSO |

### Per-service variables (`config/<service>/.env`)

Each service directory contains a `.env` file with service-specific overrides and Coolify variable references. Coolify auto-generates secrets using patterns like `${SERVICE_PASSWORD_*}` and `${SERVICE_USER_*}`.

## Roadmap

### Phase 1 -- Optimize compose files for Coolify

Clean up and standardize all `compose.yml` files to follow Coolify best practices: health checks, proper service URLs, restart policies, and volume management.

### Phase 2 -- Per-project setup via `project.env`

Make the entire stack reusable across multiple domains/projects by parameterizing everything through `project.env`. A new deployment should only require changing `project.env` values and deploying through Coolify.

### Phase 3 -- Shared dependencies, backups & safety

- Consolidate shared services (PostgreSQL, Redis, MinIO) into a single infrastructure stack instead of per-service instances.
- Add automated backup strategies for databases and object storage.
- Implement safety features: resource limits, alerting, and disaster recovery procedures.

## License

This project is provided as-is for personal and organizational use. See individual service licenses for their respective terms.
