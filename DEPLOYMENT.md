# ERPNext Production Deployment — erp.langmere.com

## Overview

- **Host:** erp.langmere.com
- **ERPNext Version:** v16.16.0 (latest)
- **Apps:** erpnext, hrms (HR & Payroll)
- **Proxy:** Cloudflare (SSL termination at edge)
- **Origin Port:** 6767 (HTTP)
- **Database:** MariaDB 11.8 (containerised)
- **Queue/Cache:** Redis 8.6

---

## Stack

| Service | Image | Role |
|---|---|---|
| frontend | custom-erpnext:16 | Nginx reverse proxy |
| backend | custom-erpnext:16 | Gunicorn app server |
| websocket | custom-erpnext:16 | Socket.IO server |
| queue-short | custom-erpnext:16 | Short/default job worker |
| queue-long | custom-erpnext:16 | Long/default/short job worker |
| scheduler | custom-erpnext:16 | Bench scheduler |
| db | mariadb:11.8 | Database |
| redis-cache | redis:8.6-alpine | Cache |
| redis-queue | redis:8.6-alpine | Job queue |

---

## Files

| File | Purpose |
|---|---|
| `compose.yaml` | Base services definition |
| `overrides/compose.mariadb.yaml` | MariaDB service |
| `overrides/compose.redis.yaml` | Redis services |
| `overrides/compose.noproxy.yaml` | Exposes frontend port directly (no Traefik) |
| `apps.json` | Custom image app list (erpnext + hrms) |
| `.env.prod` | Production environment variables |

---

## Environment Variables (`.env.prod`)

| Variable | Value |
|---|---|
| `ERPNEXT_VERSION` | v16.16.0 |
| `CUSTOM_IMAGE` | custom-erpnext |
| `CUSTOM_TAG` | 16 |
| `PULL_POLICY` | never (use local image) |
| `DB_PASSWORD` | *(see .env.prod)* |
| `FRAPPE_SITE_NAME_HEADER` | erp.langmere.com |
| `HTTP_PUBLISH_PORT` | 6767 |
| `UPSTREAM_REAL_IP_HEADER` | X-Forwarded-For |

---

## Build Custom Image

Run once (or after adding/updating apps):

```bash
docker build \
  --no-cache \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --secret=id=apps_json,src=apps.json \
  --tag=custom-erpnext:16 \
  --file=images/layered/Containerfile .
```

---

## Start / Restart Stack

```bash
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  up -d --force-recreate
```

---

## Create Site (first time only)

```bash
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  exec backend \
  bench new-site erp.langmere.com \
    --mariadb-user-host-login-scope='%' \
    --db-root-username=root \
    --db-root-password="<DB_PASSWORD from .env.prod>" \
    --admin-password=<YOUR_ADMIN_PASSWORD> \
    --install-app erpnext \
    --set-default
```

---

## Install Additional Apps

```bash
# HR & Payroll (already installed)
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  exec backend \
  bench --site erp.langmere.com install-app hrms
```

---

## Login

- **URL:** https://erp.langmere.com
- **Username:** `Administrator`
- **Password:** *(set via `--admin-password` during site creation)*

---

## Cloudflare Configuration

- DNS A record for `erp.langmere.com` → server IP, **Proxied (orange cloud)**
- SSL/TLS mode: **Full** (not Full Strict — origin is plain HTTP)
- Cloudflare forwards public ports 80/443 → origin port 6767
- Real client IP passed via `X-Forwarded-For` header

> **Note:** Port 6767 is not in Cloudflare's supported proxied port list. If Cloudflare proxying does not work, switch to DNS-only mode (grey cloud) or change `HTTP_PUBLISH_PORT` to a supported port such as `8080` or `8888`.

---

## Restart Stack

```bash
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  restart
```

## Stop Stack

```bash
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  down
```

## View Logs

```bash
# All services
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  logs -f

# Single service (e.g. backend)
docker compose --env-file .env.prod \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  logs -f backend
```
