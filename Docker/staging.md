# Runbook: Staging & Promotion

## Overview

All changes go through staging before production. The same image that passes
staging is promoted to production — never rebuild for production.

---

## Environments

| Environment | Purpose | Ports |
|---|---|---|
| Local | Development and structure testing | any |
| Staging | Pre-production verification | host port + 1 offset from production |
| Production | Live environment | standard ports |

---

## Deploying to Staging

### Via Komodo

1. Navigate to the staging stack in Komodo
2. Update the image tag or config as needed
3. Click **Deploy**
4. Monitor logs for errors
5. Verify the service is behaving correctly

### Via CLI (manual)

```bash
cd infra/stackname

# inject staging secrets
your-secrets-manager inject --env staging --output /dev/shm/stackname-staging.env

# deploy
docker compose -f compose.yaml -f compose.staging.yaml up -d

# clean up secrets file
rm /dev/shm/stackname-staging.env
```

---

## Verifying Staging

Before promoting to production, verify:

- [ ] Container started without errors (`docker ps`)
- [ ] Logs show no errors (`docker logs stackname-staging-app-1`)
- [ ] Service is reachable on its staging port
- [ ] Key functionality works as expected
- [ ] Environment variables are correct (`docker exec stackname-staging-app-1 env`)
- [ ] Volumes are mounted correctly (`docker exec stackname-staging-app-1 ls /app/data`)

---

## Promoting to Production

Only promote after staging has been verified. Use the **same image tag** that
was deployed to staging — do not rebuild.

### Via Komodo

1. Note the exact image tag running in staging
2. Navigate to the production stack in Komodo
3. Update the image tag to match staging
4. Click **Deploy**
5. Monitor logs
6. Verify production is healthy

### Via CLI (manual)

```bash
cd infra/stackname

# inject production secrets
your-secrets-manager inject --env production --output /dev/shm/stackname-production.env

# deploy — same image tag as staging
docker compose -f compose.yaml -f compose.production.yaml up -d

# clean up
rm /dev/shm/stackname-production.env
```

---

## Rolling Back

If production deployment fails, roll back to the previous image tag.

### Via Komodo

1. Navigate to the production stack
2. Update the image tag to the previous known-good version
3. Deploy

### Via CLI

```bash
# update production.env with previous image tag
APP_VERSION=1.0.4   # previous working version

# redeploy
docker compose -f compose.yaml -f compose.production.yaml up -d
```

---

## Image Tagging Convention

| Tag format | Use | Example |
|---|---|---|
| Semver | Stable releases | `1.2.3` |
| Git SHA | Traceability | `abc1234` |
| `latest` | Never in production | — |

Never deploy `latest` to production. Always pin to an explicit tag so rollback
is deterministic and you know exactly what is running.

---

## Environment Variable Changes

If you are changing environment variables rather than the image:

1. Update `staging.env` or the override file
2. Commit and push
3. Redeploy staging — Komodo picks up the change on next deploy
4. Verify staging
5. Apply same change to `production.env` or production override
6. Deploy production
