# Security Standards

## Users & Permissions

### OS Users

| User | UID | Purpose |
|---|---|---|
| `dockeruser` | 1000 | General user for most containers |
| `serviceA` | 1001 | Service-specific user where bind mount permissions require it |
| `serviceB` | 1002 | Service-specific user where bind mount permissions require it |

All service users are system users with no login shell and no home directory:

```bash
useradd --system --no-create-home --shell /sbin/nologin dockeruser
```

Your SSH user is added to the `docker` group to run Docker commands without sudo.
It is not added to any service user's group.

### Container Users

Every container runs as a non-root user. Most services use the general docker user:

```yaml
services:
  app:
    user: "1000:1000"
```

Services that interact with bind mounts requiring specific permissions use
a dedicated service user with a matching UID on the host.

### File Permissions

| Path type | Owner | Mode | Reason |
|---|---|---|---|
| Service data directory | service user | `0700` | Only that service |
| Config (read-only) | root | `0640` | Service reads, cannot modify |
| Logs | service user | `0750` | Service writes, group can read |
| Shared stack directory | root | `0770` | Shared group access |
| Cross-stack shared | root | `0770` | Shared group access |

Start at `0700` and open up deliberately. Never use `0777`.

### Shared Access

When multiple services need access to the same directory, use the `dockershared`
group rather than relaxing permissions:

```bash
# GID 2000
groupadd --gid 2000 dockershared
usermod -aG dockershared serviceA
usermod -aG dockershared serviceB
```

---

## Container Hardening

### Drop Capabilities

```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if needed
```

### Read-Only Filesystem

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /app/cache
```

### Network Isolation

Services only join networks they need. Internal services use an internal network
that is not reachable from outside the compose project:

```yaml
networks:
  backend:
    internal: true
```

### Resource Limits

All production services define memory limits:

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512m
```

---

## Docker Socket

The Docker socket (`/var/run/docker.sock`) grants full host access to anything
that can reach it. Treat it accordingly:

- Only the Komodo periphery agent and your CI/CD container need socket access
- The CI/CD container runs as a non-root user in the `docker` group — not as root
- Never expose the Docker socket to application containers
- Never expose the Docker socket over the network without TLS

---

## Secrets Rules

- Never commit secrets to Git
- Never write secrets to persistent disk
- Never pass secrets as Docker build args
- Never log secrets — verify debug logging does not expose env vars
- Secrets live in `/dev/shm` only during deploy, cleaned up immediately after
- Each service only receives secrets it explicitly needs

See [Secrets Management](./secrets.md) for the full secrets workflow.
