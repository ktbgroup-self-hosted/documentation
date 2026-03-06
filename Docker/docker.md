# Docker Standards & Practices

## Table of Contents

- [Stack Structure](#stack-structure)
- [Compose File Conventions](#compose-file-conventions)
- [Environment Management](#environment-management)
- [Volume Conventions](#volume-conventions)
- [Networking](#networking)
- [Container Configuration](#container-configuration)

---

## Stack Structure

### Repository Layout

All stacks live in repos grouped by relatedness to each other (i.e. network, apps, node, etc.). Each stack gets its own directory. Related services that share a lifecycle are grouped into one stack. Independent apps get their own stack.

```
infra/
├── docs/                     # this documentation
├── network/                  # caddy + cloudflare tunnel
├── dns/                      # dns resolver (critical, separate lifecycle)
├── postgres/                 # shared database infrastructure
├── actual-budget/
├── baserow/
├── documenso/
├── mazanoke/
├── n8n/
├── openproject/
├── paperless/
├── syncthing/
└── twenty/
```

### When to Group vs Separate

**Keep services in one stack if:**
- They share a network and communicate directly
- They have the same lifecycle — always started/stopped/updated together
- They are meaningless without each other

**Give a service its own stack if:**
- It is independent infrastructure others depend on (postgres, dns)
- It has a different update cadence to everything else
- A failure or misconfiguration should not affect other services
- You would want to redeploy it without touching anything else

---

## Compose File Conventions

### File Structure Per Stack

Every stack has the following files:

```
stackname/
├── compose.yaml                # base — shared service definitions
├── compose.staging.yaml        # staging overrides + project name
├── compose.production.yaml     # production overrides + project name
├── staging.env                 # non-secret staging config
├── production.env              # non-secret production config
└── .env.example                # documents ALL expected variables
```

### Base compose.yaml

The base file contains service definitions shared across all environments.
It must be fully environment-agnostic — no hardcoded ports, no environment-specific values,
no project name.

```yaml
# compose.yaml
services:
  api:
    image: my-api:${API_VERSION}
    restart: always
    networks:
      - backend
    volumes:
      - api_data:/app/data
      - ${BIND_MOUNT_BASE}/api/config:/app/config

  worker:
    image: my-worker:${API_VERSION}
    restart: always
    networks:
      - backend

networks:
  backend:

volumes:
  api_data:
```

**Rules for the base file:**
- Always set `restart: always` on every service
- Never hardcode ports — define them in override files
- Never define `LOG_LEVEL` or other environment-specific vars
- Never set the `name` field — this belongs in override files
- Use `${VAR}` interpolation for anything that changes per environment

### Environment Override Files

Override files only contain what differs from the base. The `name` field
is always defined here, never in the base file.

```yaml
# compose.production.yaml
name: stackname-production

env_file:
  - production.env

services:
  api:
    ports:
      - "3000:3000"
    environment:
      - LOG_LEVEL=warn
      - API_VERSION=${API_VERSION}
      - DATABASE_URL=${DATABASE_URL}

  worker:
    environment:
      - LOG_LEVEL=warn
      - QUEUE_NAME=${QUEUE_NAME}
```

```yaml
# compose.staging.yaml
name: stackname-staging

env_file:
  - staging.env

services:
  api:
    ports:
      - "3001:3000"
    environment:
      - LOG_LEVEL=debug
      - API_VERSION=${API_VERSION}
      - DATABASE_URL=${DATABASE_URL}

  worker:
    environment:
      - LOG_LEVEL=debug
      - QUEUE_NAME=${QUEUE_NAME}
```

**Rules for override files:**
- Always define `name` here, never in the base file
- Always reference the environment's `.env` file at the top level via `env_file`
- Explicitly map every environment variable each service needs — never inject
  the full env file at the service level
- Services only receive variables they actually need
- Secrets are referenced via `${VAR}` interpolation — their values come from
  the secrets manager at deploy time, not from the env file

### The .env.example File

Every stack must have a `.env.example` that documents all expected variables,
their source, and whether they are secret. This is the contract for the stack.

```bash
# .env.example

# ---- Injected by secrets manager at deploy time (never commit real values) ----
DATABASE_URL=          # postgres connection string
API_SECRET_KEY=        # application secret key

# ---- Non-secret config (safe to set in staging.env / production.env) ----
API_VERSION=           # image tag to deploy e.g. 1.2.3
LOG_LEVEL=             # debug | info | warn | error
BIND_MOUNT_BASE=       # base path for bind mounts e.g. /mnt/docker/stackname-production
QUEUE_NAME=            # worker queue name
```

### Running a Stack

```bash
# production
docker compose -f compose.yaml -f compose.production.yaml up -d

# staging
docker compose -f compose.yaml -f compose.staging.yaml up -d

# tear down
docker compose -f compose.yaml -f compose.production.yaml down
```

---

## Environment Management

### Separation Model

Staging and production are separated by compose project name. The `name` field
in each override file namespaces all containers, networks, and named volumes automatically.

| Resource | Staging | Production |
|---|---|---|
| Container | `stackname-staging-api-1` | `stackname-production-api-1` |
| Named volume | `stackname-staging_api_data` | `stackname-production_api_data` |
| Network | `stackname-staging_backend` | `stackname-production_backend` |
| Bind mount | `/mnt/docker/stackname-staging/` | `/mnt/docker/stackname-production/` |

### Environment Files

Each environment has one `.env` file containing non-secret configuration.
Secrets are never stored here — they are injected at deploy time by the secrets manager.

```bash
# staging.env
BIND_MOUNT_BASE=/mnt/docker/stackname-staging
API_VERSION=1.0.0-staging
QUEUE_NAME=staging-queue

# production.env
BIND_MOUNT_BASE=/mnt/docker/stackname-production
API_VERSION=1.0.0
QUEUE_NAME=production-queue
```

### Variable Precedence

Understanding where a variable's value comes from:

1. **Secrets manager** — injected at deploy time via pre-deploy script, highest priority
2. **`environment:` block** in compose file — explicit per-service mapping
3. **`env_file:` at top level** of override file — non-secret config interpolation
4. **Base `compose.yaml`** — structural defaults only

---

## Volume Conventions

### Named Volumes

Use named volumes for persistent application data. Docker automatically prefixes
them with the project name so they are isolated per environment with no extra work.

```yaml
volumes:
  api_data:      # becomes stackname-production_api_data automatically
  db_data:
```

### Bind Mounts

Use bind mounts only when you need direct host filesystem access — config files,
logs handled outside Docker, or data that needs to be accessed without a container.

All bind mounts live under `/mnt/docker/` on the host, organised by project name:

```
/mnt/docker/
├── stackname-production/
│   ├── api/
│   │   └── config/
│   └── worker/
└── stackname-staging/
    ├── api/
    │   └── config/
    └── worker/
```

The base path is always controlled by `BIND_MOUNT_BASE` in the environment file:

```yaml
volumes:
  - ${BIND_MOUNT_BASE}/api/config:/app/config:ro
```

**Rules for bind mounts:**
- Always use `BIND_MOUNT_BASE` — never hardcode host paths in compose files
- Mount config as read-only (`:ro`) wherever possible
- Directories must be created via Ansible before deploying — Docker will create
  them as root if they don't exist, causing permission issues
- Logs go to `/var/log/stackname/` not under `/mnt/docker/`

### Shared Directories

When multiple services in a stack need access to the same bind mount path,
use a shared group rather than relaxing permissions:

```
/mnt/docker/stackname-production/shared/   # owned by root, group dockershared, mode 0770
```

Services that need access run as users in the `dockershared` group.

---

## Networking

### Inter-Stack Networking

Stacks that need to communicate use a shared external network. The network stack
owns and creates the proxy network. All other stacks reference it as external.

**In the network stack:**
```yaml
networks:
  proxy:
    name: proxy
```

**In app stacks:**
```yaml
networks:
  proxy:
    external: true
    name: proxy
```

### Internal Networks

Services within a stack that should not be reachable from other stacks use
an internal network:

```yaml
networks:
  backend:
    internal: true   # not reachable outside this compose project
```

### Network Naming Convention

| Network | Owner | Purpose |
|---|---|---|
| `proxy` | network stack | reverse proxy to all web-accessible services |
| `stackname_backend` | each stack | internal service communication |

---

## Container Configuration

### Restart Policy

Every service must define `restart: always`. No exceptions.

```yaml
services:
  api:
    restart: always
```

### Running as Non-Root

Every container must run as a non-root user. Most services use the general
docker user (UID 1000). Services with bind mount permission requirements
use a dedicated service user.

```yaml
services:
  api:
    user: "1000:1000"   # general docker user for most services
```

For services needing specific bind mount access, the UID is defined in
Ansible vars and applied consistently across all servers.

### Dropping Capabilities

Drop all Linux capabilities and add back only what is needed:

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if binding to port < 1024
```

### Read-Only Filesystem

Use read-only root filesystems where possible:

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
      - /app/cache
```

### Resource Limits

Define memory limits on all production services to prevent a runaway container
from affecting other services on the same host:

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          memory: 512m
```
