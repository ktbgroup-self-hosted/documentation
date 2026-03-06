# Networking

## Ingress Flow

```
Internet
    │
    ▼
Cloudflare (DNS + Tunnel)
    │
    ▼
Caddy (Reverse Proxy)
    │
    ├──▶ actual-budget
    ├──▶ baserow
    ├──▶ n8n
    ├──▶ paperless
    └──▶ ... (other web-accessible services)

DNS Resolver (internal)
    │
    ├──▶ Caddy
    └──▶ Internal services
```

## Networks

### proxy (external, shared)

Owned and created by the `network` stack. All services that need to be
reachable via Caddy join this network.

```yaml
# In the network stack — creates the network
networks:
  proxy:
    name: proxy

# In app stacks — references the existing network
networks:
  proxy:
    external: true
    name: proxy
```

### stackname_backend (internal per stack)

Created by each stack for internal service communication. Not reachable
from other stacks. Used for service-to-database communication within a stack.

```yaml
networks:
  backend:
    internal: true
```

## Inter-Stack Dependencies

Some stacks depend on others being up first. Boot order matters after a full
server restart.

| Stack | Depends On |
|---|---|
| All web apps | network stack (proxy network must exist) |
| documenso | postgres |
| openproject | postgres |
| twenty | postgres |
| All stacks | dns |

**Safe boot order:**
1. dns
2. postgres
3. network (caddy + cloudflare)
4. Everything else

## Port Conventions

Production services behind Caddy do not expose host ports — Caddy reaches them
via the proxy network directly. Only services not behind Caddy expose host ports.

For staging, ports are offset by 1 from the production port to allow both
environments to run on the same host without collision:

| Service | Production port | Staging port |
|---|---|---|
| api | 3000 | 3001 |
| app | 8080 | 8081 |

Services behind Caddy in production may still expose a port in staging
for direct access during testing.
