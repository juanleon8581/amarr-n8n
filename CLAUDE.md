# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Dockerized n8n automation instance. Goal: configure locally, then deploy to a VPS. Not a software project — no build pipeline, no tests, no linter.

## Common Commands

```bash
# Start (detached)
docker compose up -d

# Stop
docker compose down

# View logs
docker compose logs -f n8n

# Restart after .env change
docker compose down && docker compose up -d

# Check health
docker compose ps
```

## Architecture

Single-service Docker Compose stack:

- **n8n** — `n8nio/n8n:latest`, exposed on port `5678`
- **n8n_data** — named Docker volume mounted at `/home/node/.n8n` (stores workflows, credentials, DB)
- **.env** — all runtime config (host, auth, encryption key, timezone)

Basic auth is enabled via `N8N_BASIC_AUTH_*` env vars. Webhooks reach n8n through `WEBHOOK_URL`.

## Critical Config Rules

- `N8N_ENCRYPTION_KEY` encrypts stored credentials. **Never change it after workflows/credentials exist** — doing so breaks all stored credentials.
- Before exposing to internet, change `N8N_BASIC_AUTH_PASSWORD` from `changeme`.
- For VPS deploy: update `N8N_HOST`, `N8N_PROTOCOL`, and `N8N_WEBHOOK_URL` to the public domain/IP. If using HTTPS, set `N8N_PROTOCOL=https`.

## VPS Deploy Checklist

1. Copy project files to VPS (`.env` + `docker-compose.yml`)
2. Set production values in `.env` (host, protocol, webhook URL, strong password)
3. Ensure port `5678` is open (or put a reverse proxy in front)
4. `docker compose up -d`
5. Verify at `http://<host>:5678/healthz`
