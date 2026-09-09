# Coolify Deployment Guide

Deploy Cal.diy on Coolify using Docker Compose with separate web and API domains.

## Prerequisites

- Coolify instance with Git-based Docker Compose deployment
- Two DNS records pointing to your Coolify server:
  - `cal.jasperfilmz.online` → `144.24.148.13`
  - `api.cal.jasperfilmz.online` → `144.24.148.13`
- Coolify proxy owns public HTTP/HTTPS ports (80/443)

## Coolify Configuration

| Field | Value |
|-------|-------|
| Resource type | Docker Compose |
| Repository | This fork |
| Branch | `main` |
| Compose file | `docker-compose.coolify.yml` |
| Base directory | `/` |

## Coolify Domain Routing

Configure two domains in the Coolify UI:

| Domain | Target service | Internal port |
|--------|---------------|---------------|
| `cal.jasperfilmz.online` | `calcom` | `3000` |
| `api.cal.jasperfilmz.online` | `calcom-api` | `5555` |

Coolify will route each domain to the corresponding container's internal port. No host port bindings are required.

## Required Environment Variables

Create these in the Coolify UI. Use `env.coolify.example` as a template.

### Required Secrets

| Variable | How to generate |
|----------|-----------------|
| `NEXTAUTH_SECRET` | `openssl rand -base64 32` |
| `CALENDSO_ENCRYPTION_KEY` | `openssl rand -base64 24` |
| `POSTGRES_PASSWORD` | Any strong random password |
| `JWT_SECRET` | `openssl rand -base64 32` |

### Required Configuration

| Variable | Value |
|----------|-------|
| `NEXT_PUBLIC_WEBAPP_URL` | `https://cal.jasperfilmz.online` |
| `NEXTAUTH_URL` | `https://cal.jasperfilmz.online/api/auth` |
| `NEXT_PUBLIC_API_V2_URL` | `https://api.cal.jasperfilmz.online/api/v2` |
| `API_URL` | `https://api.cal.jasperfilmz.online` |
| `WEB_APP_URL` | `https://cal.jasperfilmz.online` |
| `POSTGRES_USER` | `calcom` |
| `POSTGRES_DB` | `calendso` |
| `DATABASE_URL` | `postgresql://calcom:<POSTGRES_PASSWORD>@database:5432/calendso` |
| `DATABASE_DIRECT_URL` | Same as `DATABASE_URL` |
| `ALLOWED_HOSTNAMES` | `"cal.jasperfilmz.online","api.cal.jasperfilmz.online"` |

### Optional (have safe defaults in compose)

| Variable | Default |
|----------|---------|
| `REDIS_URL` | `redis://redis:6379` |
| `CALCOM_TELEMETRY_DISABLED` | `1` |
| `CSP_POLICY` | `non-strict` |
| `NODE_ENV` | `production` |
| `API_PORT` | `5555` |
| `REWRITE_API_V2_PREFIX` | `1` |

## Internal Architecture

```
Internet
  ↓
Coolify Proxy (80/443)
  ├── cal.jasperfilmz.online ──→ calcom:3000 (Next.js web)
  └── api.cal.jasperfilmz.online ──→ calcom-api:5555 (NestJS API v2)
                                      ↓
                                  database:5432 (PostgreSQL)
                                  redis:6379 (Redis)
```

### Service Names and Internal Ports

| Service | Container name | Internal port | Host port |
|---------|---------------|---------------|-----------|
| Web | `calcom` | `3000` | None |
| API v2 | `calcom-api` | `5555` | None |
| Database | `database` | `5432` | None |
| Redis | `redis` | `6379` | None |

No host ports are required. Coolify routes each domain to the appropriate service through its internal proxy.

## CORS and API Access

API v2 enables CORS with `origin: "*"`, so the browser can call `https://api.cal.jasperfilmz.online/api/v2` directly from any origin. Next.js also proxies `/api/v2/*` requests server-side through rewrites configured by `NEXT_PUBLIC_API_V2_URL`.

## Database Migrations

Migrations run automatically at container startup via `scripts/start.sh`:

```sh
npx prisma migrate deploy --schema /calcom/packages/prisma/schema.prisma
npx ts-node --transpile-only /calcom/scripts/seed-app-store.ts
```

No manual migration step is required for initial deployment or updates.

## Deployment Steps

1. Push this branch to your Git repository.
2. In Coolify, create a new **Docker Compose** application.
3. Set the repository, branch, and compose file (`docker-compose.coolify.yml`).
4. Configure the two domains in Coolify:
   - `cal.jasperfilmz.online`
   - `api.cal.jasperfilmz.online`
5. Enter the environment variables from above.
6. Deploy.

## Verification

After deployment:

```bash
# Check container status
docker compose -f docker-compose.coolify.yml ps

# View web app logs
docker compose -f docker-compose.coolify.yml logs calcom

# View API v2 logs
docker compose -f docker-compose.coolify.yml logs calcom-api

# Test API v2 health (from inside the stack)
docker compose -f docker-compose.coolify.yml exec calcom-api wget -q -O- http://localhost:5555/health
```

Open `https://cal.jasperfilmz.online` in a browser. The first visit should show the setup wizard.

## Troubleshooting

### Build fails with database connection error

The Next.js build requires a database connection during `docker compose build`. Coolify builds images before starting containers. If the build fails:

1. Ensure the `database` service is defined in the same compose file (it is).
2. Check Coolify build logs for the exact error.
3. If needed, pre-pull the Postgres image: `docker compose -f docker-compose.coolify.yml pull database`
4. As a last resort, set `DOCKER_BUILDKIT=0` in Coolify's build environment.

### Next-auth CLIENT_FETCH_ERROR

If you see `CLIENT_FETCH_ERROR` in logs, ensure `NEXTAUTH_URL` matches your public domain. The compose defaults it to `${NEXT_PUBLIC_WEBAPP_URL}/api/auth`.

### API domain returns 404

Ensure Coolify has the `api.cal.jasperfilmz.online` domain configured and pointing to the `calcom-api` service on internal port `5555`.

## Updating

1. Pull the latest code in Coolify.
2. Redeploy.
3. Migrations run automatically on startup.

## Backup

Backup the PostgreSQL volume:

```bash
docker compose -f docker-compose.coolify.yml exec database pg_dump -U calcom calendso > backup.sql
```

To restore:

```bash
cat backup.sql | docker compose -f docker-compose.coolify.yml exec -T database psql -U calcom calendso
```

## Intentionally Excluded

- **No host port bindings** — Coolify proxy handles external routing.
- **No Prisma Studio** — not needed for production deployment.
- **No Redis/PostgreSQL host exposure** — both are internal-only.
- **No nginx/Traefik** — Coolify owns the public reverse proxy.
- **No monitoring stack** — keep it lightweight.
