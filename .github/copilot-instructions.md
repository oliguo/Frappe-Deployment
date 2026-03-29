# Frappe-Deployment — Copilot Instructions

## Project Overview

Docker-based all-in-one Frappe v16 deployment stack on Ubuntu 22.04.
Bundles: ERPNext, CRM, HRMS, Payments, Drive, HelpDesk, Raven, frappe_whatsapp.
Infrastructure: Traefik v3 (TLS/reverse-proxy) + MariaDB 10.6 + Redis + phpMyAdmin.

## Repository Layout

```
frappe_files/        # ← Source of truth: apps.json and pwd-custom.yml to copy into docker/
traefik3/            # Traefik v3 compose + config (manages ports 80/443)
phpmyadmin-5.2/      # phpMyAdmin compose (port 8900)
docker/              # Cloned frappe/frappe_docker — do NOT edit upstream files here
screens_ref/         # Screenshot references only
```

> Key principle: `frappe_files/` holds the customisations. After cloning `frappe/frappe_docker` into `docker/`, copy files from `frappe_files/` into `docker/`.

## Critical Configuration Points

| File | What to customise |
|------|-------------------|
| `frappe_files/pwd-custom.yml` | Domain (`sub.yourdomain.com`), DB root password (`admin` default) |
| `traefik3/traefik.yml` | ACME email (`your@email.com`) |
| `frappe_files/apps.json` | App URLs and branches |

**Passwords**: `MYSQL_ROOT_PASSWORD` / `MARIADB_ROOT_PASSWORD` default to `admin` — change before production.

## Docker Network

All services share `frappe_network` (external). Create it once before starting anything:
```bash
sudo docker network create frappe_network
```
Traefik uses `frappe_network`; frappe services use `frappe_docker_network` internally (defined in `pwd-custom.yml`).

## Deployment Workflow

```bash
# 1. Infrastructure (run once)
sudo docker network create frappe_network
cd traefik3 && sudo docker compose up -d
cd ../phpmyadmin-5.2 && sudo docker compose up -d

# 2. Clone upstream frappe_docker
cd ~ && git clone https://github.com/frappe/frappe_docker frappe/docker

# 3. Copy customisations
cp ~/frappe/frappe_files/* ~/frappe/docker/

# 4. Build custom image (Frappe v15 base + all apps baked in)
cd ~/frappe/docker
mkdir -pv db-data logs sites redis-queue-data
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)
sudo docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=frappe-all:1.0.0 \
  --file=images/layered/Containerfile .

# 5. Start stack
sudo docker compose -f pwd-custom.yml up -d
```

## App Branches (apps.json)

| App | Branch |
|-----|--------|
| frappe/erpnext | version-16 |
| frappe/payments | version-15 |
| frappe/hrms | version-16 |
| frappe/crm | main |
| frappe/helpdesk | main |
| frappe/drive | main |
| The-Commit-Company/raven | main |
| shridarpatil/frappe_whatsapp | master |

> `payments` is pinned to version-15 — verify compatibility when upgrading ERPNext/HRMS.

## One-Shot Site Creation

`create-site` container runs once at startup and installs all apps on the `frontend` site:
- Default admin password: `admin`
- DB root password: `admin`
- Both must be changed for production

Normal shutdown containers: `configurator` and `create-site` — they exit after completing their work.

## Tear-Down / Reset

```bash
sudo docker compose -f pwd-custom.yml down
sudo docker volume prune -a
sudo rm -rf db-data logs sites redis-queue-data
```

## Common Pitfalls

- **`frappe_network` must exist** before starting Traefik or phpMyAdmin.
- **Domain labels in `pwd-custom.yml`**: update both the `http` and `https` router rules for the `frontend` service.
- **ACME email** in `traefik3/traefik.yml` must be real for Let's Encrypt to work.
- **`create-site` loop**: if it keeps restarting, check `db:3306` is reachable and `common_site_config.json` exists with correct keys.
- **`base64 -w 0`** flag required on Linux; macOS uses `base64` without `-w`.
- **phpMyAdmin `external_links`**: connects to `frappe_docker-db-1` — container name must match your Docker project name.

## Useful Commands

```bash
# Tail create-site logs
sudo docker logs -f frappe_dev-create-site-1

# Restart a specific service
sudo docker compose -f pwd-custom.yml restart backend

# Check all running containers
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## Reference

- Full setup instructions: [README.md](../README.md)
- Frappe Docker upstream docs: `docker/docs/`
