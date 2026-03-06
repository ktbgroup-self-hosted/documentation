# Runbook: Adding a New Stack

## Overview

This runbook covers the full process of adding a new service to the homelab,
from creating the compose files through to deploying in Komodo.

---

## Step 1: Create the Directory Structure

```bash
mkdir -p infra/stackname
cd infra/stackname
```

## Step 2: Create the Base Compose File

Create `compose.yaml` with the service definition. Keep it fully environment-agnostic.

```yaml
# compose.yaml
services:
  app:
    image: someimage:${APP_VERSION}
    restart: always
    networks:
      - backend
    volumes:
      - app_data:/app/data
      - ${BIND_MOUNT_BASE}/app/config:/app/config:ro

networks:
  backend:

  # include if this service needs to be reachable via caddy
  proxy:
    external: true
    name: proxy

volumes:
  app_data:
```

## Step 3: Create Environment Override Files

```yaml
# compose.production.yaml
name: stackname-production

env_file:
  - production.env

services:
  app:
    ports:
      - "3000:3000"        # only if not behind caddy
    environment:
      - LOG_LEVEL=warn
      - APP_VERSION=${APP_VERSION}
      - DATABASE_URL=${DATABASE_URL}
```

```yaml
# compose.staging.yaml
name: stackname-staging

env_file:
  - staging.env

services:
  app:
    ports:
      - "3001:3000"        # different port to avoid collision with production
    environment:
      - LOG_LEVEL=debug
      - APP_VERSION=${APP_VERSION}
      - DATABASE_URL=${DATABASE_URL}
```

## Step 4: Create Environment Files

```bash
# production.env
BIND_MOUNT_BASE=/mnt/docker/stackname-production
APP_VERSION=1.0.0

# staging.env
BIND_MOUNT_BASE=/mnt/docker/stackname-staging
APP_VERSION=1.0.0-staging
```

## Step 5: Create .env.example

Document every variable the stack expects:

```bash
# .env.example

# ---- Secrets (injected by secrets manager) ----
DATABASE_URL=          # postgres connection string

# ---- Non-secret config ----
APP_VERSION=           # image tag e.g. 1.0.0
LOG_LEVEL=             # debug | warn
BIND_MOUNT_BASE=       # base path for bind mounts
```

## Step 6: Add Bind Mount Directories via Ansible

Add the bind mount directories to your Ansible provisioning playbook.
Docker will create them as root if they don't exist, causing permission issues.

```yaml
# In your Ansible playbook
- name: Create bind mount directories for stackname
  file:
    path: "{{ item.path }}"
    owner: "{{ item.uid }}"
    group: "{{ item.uid }}"
    mode: "{{ item.mode }}"
    state: directory
  loop:
    - { path: /mnt/docker/stackname-production/app/config, uid: 1000, mode: "0750" }
    - { path: /mnt/docker/stackname-staging/app/config, uid: 1000, mode: "0750" }
```

Run the playbook against the target server before deploying.

## Step 7: Add Secrets to Secrets Manager

Add any required secrets to your secrets manager under the appropriate
path for staging and production.

## Step 8: Test Locally

Verify the compose structure works before deploying to a server:

```bash
# create local test directories
mkdir -p ~/docker/stackname-staging/app/config

# spin up staging
docker compose -f compose.yaml -f compose.staging.yaml up -d

# verify container names and prefix
docker ps

# verify env vars are correct
docker exec stackname-staging-app-1 env

# verify bind mount
docker exec stackname-staging-app-1 ls /app/config

# tear down
docker compose -f compose.yaml -f compose.staging.yaml down
```

## Step 9: Add to Komodo

Create two stacks in Komodo — one for staging, one for production.

**Staging stack:**
- Name: `stackname-staging`
- Repo: your infra repo
- Run directory: `stackname`
- Compose files: `compose.yaml`, `compose.staging.yaml`
- Pre-deploy script: secrets injection for staging

**Production stack:**
- Name: `stackname-production`
- Repo: your infra repo
- Run directory: `stackname`
- Compose files: `compose.yaml`, `compose.production.yaml`
- Pre-deploy script: secrets injection for production

## Step 10: Deploy Staging First

Deploy to staging and verify everything works before touching production.
See [Staging & Promotion runbook](./staging.md) for the full process.

---

## Checklist

- [ ] Directory created under `infra/`
- [ ] `compose.yaml` created — environment agnostic, no hardcoded values
- [ ] `compose.staging.yaml` created — `name`, `env_file`, per-service env mapping
- [ ] `compose.production.yaml` created — `name`, `env_file`, per-service env mapping
- [ ] `staging.env` created — non-secret staging config
- [ ] `production.env` created — non-secret production config
- [ ] `.env.example` created — all variables documented with sources
- [ ] Bind mount directories added to Ansible playbook and provisioned
- [ ] Secrets added to secrets manager
- [ ] Tested locally with alpine or actual image
- [ ] Komodo staging stack created
- [ ] Komodo production stack created
- [ ] Deployed to staging and verified
- [ ] Promoted to production
- [ ] Stack documented in architecture diagram
