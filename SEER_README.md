# Seer Deployment Guide

This documents the Seer-specific deployment of Attendee on AWS EC2 behind an ALB.

## Architecture

- **EC2 instance** running Docker containers
- **AWS ALB** for SSL termination and load balancing
- **Domain**: `meetingbot-dev.getseer.dev` (routed via ALB target group)
- **RDS PostgreSQL** for the database
- **Redis** runs as a container alongside the app

## Quick Start

```bash
# Start all services
docker compose -f seer.docker-compose.yml up -d

# View logs
docker compose -f seer.docker-compose.yml logs -f

# Restart after config changes
docker compose -f seer.docker-compose.yml down && docker compose -f seer.docker-compose.yml up -d
```

## Configuration

### Settings Module

Seer uses a dedicated Django settings file: `attendee/settings/seer_production.py`

This is set via `DJANGO_SETTINGS_MODULE=attendee.settings.seer_production` in `.env`.

Key differences from the default `production.py`:
- `SECURE_PROXY_SSL_HEADER` is always enabled (ALB sends `X-Forwarded-Proto`)
- No `SECURE_SSL_REDIRECT` (ALB handles HTTPS redirection)
- `CSRF_TRUSTED_ORIGINS` is configured for the ALB domain

### Environment Variables (`.env`)

| Variable | Description | Example |
|---|---|---|
| `DJANGO_SETTINGS_MODULE` | Must be `attendee.settings.seer_production` | `attendee.settings.seer_production` |
| `DJANGO_SECRET_KEY` | Django secret key | (generate a secure random string) |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/db` |
| `POSTGRES_SSL_REQUIRE` | Require SSL for DB connection | `true` / `false` |
| `REDIS_URL` | Redis connection string | `redis://redis:6379/5` |
| `CSRF_TRUSTED_ORIGINS` | Comma-separated trusted origins | `https://meetingbot-dev.getseer.dev` |
| `AWS_RECORDING_STORAGE_BUCKET_NAME` | S3 bucket for recordings | `seer-meetingbot-dev` |
| `AWS_AUDIO_CHUNK_STORAGE_BUCKET_NAME` | S3 bucket for audio chunks | `seer-meetingbot-dev` |
| `AWS_ACCESS_KEY_ID` | AWS access key | |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | |
| `AWS_DEFAULT_REGION` | AWS region | `us-east-1` |
| `DJANGO_SUPERUSER_EMAIL` | Admin login email | |
| `DJANGO_SUPERUSER_PASSWORD` | Admin login password | |

### Docker Compose

Use `seer.docker-compose.yml` (not the default `docker-compose.yml`).

Services:
- **redis** — Redis server with persistent storage
- **attendee-init** — Runs migrations, collectstatic, and creates superuser
- **attendee-web** — Gunicorn serving the Django app on port 8000
- **attendee-worker** — Celery worker for async tasks
- **attendee-scheduler** — Task scheduler

### AWS ALB Setup

- ALB listener on port 443 (HTTPS) forwarding to target group on port 8000 (HTTP)
- Health check path: `/api/v1/` or any valid endpoint
- The ALB handles SSL termination; traffic between ALB and EC2 is HTTP
- `SECURE_PROXY_SSL_HEADER` in Django settings ensures it recognizes forwarded HTTPS requests

## Troubleshooting

### CSRF 403 Error
If you see "CSRF verification failed" when accessing through the domain, ensure `CSRF_TRUSTED_ORIGINS` in `.env` includes your domain with the `https://` prefix.

### Static Files Not Loading
Run `docker compose -f seer.docker-compose.yml restart attendee-init` to re-collect static files.
