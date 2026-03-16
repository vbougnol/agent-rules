---
trigger: model_decision
scope: deployment
---

# Deployment and Local Development Workflow

> Rules and procedures for deploying services locally and managing the service startup cascade.
> Applies when setting up local development, Docker Compose, or deployment pipelines.

---

## 1. Architecture Overview

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐     ┌───────────┐
│   Frontend   │────>│   API Gateway   │────>│     Backend      │────>│ Database  │
│  (nginx/dev) │     │  (Fastify/proxy)│     │  (Fastify/Prisma)│     │ (Postgres)│
└──────────────┘     └─────────────────┘     └──────────────────┘     └───────────┘
```

---

## 2. Startup Cascade (enforced by depends_on + healthchecks)

Services must start in this order:

1. **Database** starts first
   - Healthcheck: `pg_isready -U postgres`

2. **Backend** starts after database is healthy
   - Runs Prisma migrations on startup (`prisma migrate deploy`)
   - Healthcheck: `GET /healthcheck`

3. **API Gateway** starts after backend is healthy
   - Checks backend connectivity
   - Healthcheck: `GET /healthcheck`

4. **Frontend** starts after gateway is healthy
   - Verifies gateway is reachable
   - Serves static assets

---

## 3. Docker Compose Local Deployment

### Launch command
```bash
docker compose -f infrastructure/compose.yaml -f infrastructure/compose.yaml.local up -d --build
```

### Service communication (Docker DNS)

| Service | Internal URL (Docker) | Host URL |
|---------|----------------------|----------|
| Database | `<project>-postgres:5432` | `localhost:5432` |
| Backend | `<project>-backend:8080` | `localhost:8081` |
| API Gateway | `<project>-api-gateway:8080` | `localhost:8080` |
| Frontend | `<project>-frontend:80` | `localhost:80` |

**IMPORTANT:** Internal Docker URLs use the container port, not the host port.

### Environment files

| Service | Config | Secrets (not committed) |
|---------|--------|------------------------|
| Backend | `.env` | `.env.local` |
| API Gateway | `.env` | `.env.local` |
| Frontend | `.env` | N/A |
| Database | `.env` | N/A |

---

## 4. Local Development (without Docker)

### Start each service manually
```bash
# Terminal 1: Database (must be running)
# Terminal 2: Backend
cd src/servers/<project>-backend && npm run dev

# Terminal 3: API Gateway
cd src/servers/<project>-api-gateway && npm run dev

# Terminal 4: Frontend
cd src/servers/<project>-frontend && npm run dev
```

### Always run the full stack locally
Do not bypass the API Gateway for local development. The gateway provides:
- CORS handling
- Security headers
- Request ID injection
- Rate limiting

---

## 5. Environment Configuration

### Frontend
```bash
# API base path (optional, default: /api)
VITE_API_BASE_PATH=/api
```

### API Gateway
```bash
PORT=3000
HOST=0.0.0.0
BACKEND_URL=http://localhost:3001        # Local dev
# BACKEND_URL=http://<project>-backend:8080  # Docker
```

### Backend
```bash
PORT=3001
HOST=0.0.0.0
DATABASE_URL=postgresql://user:pass@localhost:5432/db
```

---

## 6. Troubleshooting

### Frontend shows ERR_CONNECTION_REFUSED
- Check: `docker ps -a` — all containers should be `Up (healthy)`
- If backend is `starting`, wait for Prisma migration to complete (~60s)

### Backend container keeps restarting
- Check: `docker logs <project>-backend`
- Common causes: OIDC provider unreachable, duplicate route crash, migration failure

### Full stack cleanup
```bash
docker compose -f infrastructure/compose.yaml -f infrastructure/compose.yaml.local down -v
docker compose -f infrastructure/compose.yaml -f infrastructure/compose.yaml.local up -d --build
```

### Database drift
- If `prisma migrate dev` complains about drift: `prisma migrate reset` (local only, destroys data)

---

## 7. Deployment Rules

- **Never** deploy with `prisma db push` — always use `prisma migrate deploy`
- **Never** deploy without running the baseline first
- **Always** verify health endpoints respond after deployment
- **Always** check that the startup cascade completes successfully
- **Always** verify gateway proxy is working (frontend can reach backend through gateway)
- Log `[STARTUP]` checkpoints at each stage for observability

---

## 8. Post-Deployment Verification Checklist

After every deployment, verify:

```
1. [ ] All containers/services are healthy: docker ps -a shows "Up (healthy)"
2. [ ] Health endpoints respond:
       - Backend:  GET /healthcheck → 200
       - Gateway:  GET /healthcheck → 200
       - Frontend: loads in browser
3. [ ] Gateway proxy works: frontend can call /api/* through gateway
4. [ ] Authentication flow works: login → token → protected call → success
5. [ ] Database migrations applied: no pending migrations in logs
6. [ ] Structured logs are flowing: check log output for expected events
7. [ ] No error spikes: monitor error rate for first 5 minutes
```

---

## 9. Rollback Procedure

### Fast rollback (< 5 minutes)

```bash
# 1. Revert to previous image/commit
docker compose -f infrastructure/compose.yaml down
git checkout <previous-tag>
docker compose -f infrastructure/compose.yaml up -d --build

# 2. Verify health
curl http://localhost:8080/healthcheck
```

### Database rollback

**Additive migrations** (new table, new optional column):
- No DB rollback needed — the old code simply ignores the new fields

**Destructive migrations** (column removal, rename, type change):
- Must have a reverse migration SQL prepared BEFORE deploying
- Apply reverse migration manually then redeploy old code
- This is why destructive migrations require ADR + rollback plan

### Rollback decision tree

| Situation | Action |
|-----------|--------|
| Error rate > 5% after deploy | Immediate rollback |
| Single feature broken, others OK | Feature flag off if possible, else rollback |
| DB migration failed mid-way | Stop, assess, manual fix or reset |
| Performance degradation > 50% | Rollback and investigate |
