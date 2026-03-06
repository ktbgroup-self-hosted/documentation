# Secrets Management

## Overview

Secrets are never stored in Git, never written to persistent disk in plaintext,
and never defined in compose files directly. They are injected at deploy time
by the secrets manager and passed to containers as environment variables.

## How It Works

```
Secrets Manager
      │
      ▼
Pre-deploy script runs
      │
      ▼
Secrets written to /dev/shm/stackname.env   ← tmpfs, never touches disk
      │
      ▼
docker compose up reads secrets via env interpolation
      │
      ▼
Container receives only the vars explicitly mapped to it
      │
      ▼
/dev/shm file removed after deploy
```

## /dev/shm

`/dev/shm` is a tmpfs mount — it exists only in memory and is never written
to disk. This is the only acceptable location for temporary secret files.

**Never write secret files to:**
- `/tmp` — may be written to disk depending on system config
- `/mnt/docker/` — persistent storage, wrong place for secrets
- Any path inside the repo

## What Goes Where

| Type | Location | Example |
|---|---|---|
| Secrets | Secrets manager → `/dev/shm` at deploy time | `DATABASE_URL`, `API_KEY` |
| Non-secret config | `staging.env` / `production.env` in repo | `API_VERSION`, `LOG_LEVEL` |
| Structural vars | `staging.env` / `production.env` in repo | `BIND_MOUNT_BASE` |

## Referencing Secrets in Compose Files

Secrets are referenced via interpolation in the service's `environment:` block.
The value is never defined in the compose file — it comes from the injected env file at runtime.

```yaml
# compose.production.yaml
services:
  api:
    environment:
      - DATABASE_URL=${DATABASE_URL}    # value injected by secrets manager
      - API_KEY=${API_KEY}              # value injected by secrets manager
      - LOG_LEVEL=warn                  # value defined inline, not secret
      - API_VERSION=${API_VERSION}      # value from production.env, not secret
```

## The .env.example Contract

Every stack must have a `.env.example` documenting all variables, their source,
and whether they are secret. This is the single reference for what a stack needs
to run.

```bash
# .env.example

# ---- Secrets (injected by secrets manager, never commit real values) ----
DATABASE_URL=
API_SECRET_KEY=
SMTP_PASSWORD=

# ---- Non-secret config (defined in staging.env / production.env) ----
API_VERSION=
LOG_LEVEL=
BIND_MOUNT_BASE=
```

## Rules

- Never commit a real secret to Git under any circumstances
- Never log secrets — ensure `LOG_LEVEL=debug` does not expose env vars in output
- Never pass secrets as Docker build args — they end up in image layers
- Every secret a service receives must be explicitly mapped in its `environment:` block
- Services only receive secrets they actually need — a worker should not receive
  a secret only the API needs
- Always clean up `/dev/shm` files after deploy
