---
trigger: model_decision
scope: infrastructure
---

# Security, Dependencies, and Service Startup

> Rules for dependency management, secrets handling, service bootstrap, and operational security.
> Applies when touching: package.json, auth/authz, admin endpoints, startup logic, or observability.

---

## 1. Dependency Policy

### No new dependency without explicit human approval

Before adding any dependency:

1. **Check existing deps** — does the repo already have something suitable?
2. **Check standard library** — can this be done without a dependency?
3. **Justify the need** — why are existing code/deps insufficient?
4. **Verify license** — compatible with project license?
5. **Verify maintenance health** — actively maintained? Last release? Open issues?
6. **Update lockfiles deterministically** — no manual lockfile edits
7. **ADR required** — document the decision for non-trivial additions

**If uncertain: do not add the dependency. Ask.**

### Preferred order:
1. Existing approved dependencies in the repo
2. Standard library / built-in capabilities
3. Widely adopted, well-maintained libraries

---

## 2. Secrets Management

### Hard rules:

- **Never** hardcode credentials, tokens, or API keys in source code
- **Never** log secrets (passwords, tokens, API keys, connection strings)
- **Never** commit `.env` files with real secret values
- **Always** use environment variables or secret managers
- **Always** use `.env.example` or `.env.template` with placeholder values

### Examples:
```typescript
// FORBIDDEN
const API_KEY = 'sk-abc123xyz789';
const DB_URL = 'postgresql://user:password@host:5432/db';

// MANDATORY
const API_KEY = process.env.API_KEY;
const DB_URL = process.env.DATABASE_URL;
```

---

## 3. Authentication & Authorization

### Core principles:
- **AuthN is not AuthZ** — authentication (identity) and authorization (permissions) are separate concerns
- **Deny by default** — when unclear, deny access
- **Fail closed** — sensitive operations must fail safely on error
- **Sanitized errors** — clients get generic errors; detailed context goes to logs (without secrets)

### Endpoint protection:

| Endpoint Type | Protection |
|---------------|-----------|
| Public routes | No auth required (login, health, public API) |
| User routes | `Authorization: Bearer <token>` required |
| Admin routes | Under `/api/admin/*` with explicit RBAC |
| Internal routes | Under `/internal/*`, hidden/protected by default |

### Admin surface rules:
- Ordinary users must receive `403` for admin-only behavior
- Tenant scoping must come from auth context/token, not request body
- Non-read admin actions must generate audit logs

### Internal endpoints:
- `/internal/*` endpoints are closed by default
- Must not be exposed publicly by accident
- Protected by internal token or network policy

---

## 4. Gateway Security Headers

When touching gateway response flow, do **not** remove these defaults:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: no-referrer
```

These are part of the security contract. Removing them requires ADR justification.

---

## 5. Service Startup Protocol

### Startup cascade

Services must start in dependency order:
```
Database → Backend → API Gateway → Frontend
```

### Startup rules:

1. **Wait for dependencies** — do not crash because a downstream service is not ready yet
2. **Active health checks** — check downstream readiness roughly every 5 seconds
3. **Log waiting state** — "still waiting for X" logs roughly every 30 seconds
4. **Explicit startup checkpoints** — log `[STARTUP]` at each stage
5. **Never silent waits** — no silent retries or silent crashes
6. **Never log secrets during config debugging**

### Health endpoints:

| Endpoint | Purpose |
|----------|---------|
| `/health` or `/healthcheck` | Liveness — is the process running? |
| `/ready` | Readiness — are critical dependencies available? |

### Timeouts everywhere:
- HTTP client calls
- Database queries
- External API calls
- Queue consumers

---

## 6. Observability Rules

### Structured logging (mandatory)

```typescript
// CORRECT — structured log
logger.info({
    event: 'http_request_completed',
    method: 'POST',
    path: '/api/items',
    statusCode: 201,
    duration: 45,
    requestId: req.id
});

// FORBIDDEN — unstructured log
console.log('Request completed');
```

### Required log fields:
- `requestId` / correlation ID
- Operation name / route
- Duration
- Status code
- Error type/code (if any)

### Observability should not regress:
- Keep request-id and startup signal quality intact when making changes
- Do not replace structured/standardized logs with ad hoc noise

---

## 7. Anti-Patterns

- Adding a package with no justification or policy check
- Hardcoding secrets in source or `.env` examples with real values
- Exposing `/internal/*` routes by default
- Trusting role/tenant identifiers from request bodies for admin authorization
- Removing gateway security headers accidentally
- Replacing startup checkpoints with silent retry loops
- Using `console.log` instead of structured logging in production
- Crashing on startup because a downstream service isn't ready yet
- Logging sensitive data (tokens, passwords, connection strings)

---

## 8. LLM / External API Guardrails

When the project integrates LLM or expensive external APIs, these guardrails are mandatory (see `12-llm-integration.md` for full details):

| Guardrail | Purpose | Default Config |
|-----------|---------|---------------|
| **Timeout** | Prevent hanging requests | 60s standard, 120s heavy |
| **Circuit Breaker** | Stop calling a failing service | Open after 3 failures, reset after 60s |
| **Rate Limiter** | Respect provider limits | Per-minute token/request budget |
| **Concurrency Limiter** | Prevent resource exhaustion | Max 5 concurrent calls |
| **Budget Guard** | Control costs | Max $/request, $/day |
| **Retry with backoff** | Handle transient errors | 3 retries, exponential backoff |

**Rule:** No external API call without at least timeout + retry + circuit breaker.

---

## 9. Monitoring and Alerting

### What to monitor

| Metric | Alert threshold | Action |
|--------|----------------|--------|
| Error rate (5xx) | > 1% over 5 min | Investigate immediately |
| Response latency P95 | > 2s for API, > 5s for LLM | Profile and optimize |
| Health endpoint failures | Any failure | Check service health |
| LLM cost per day | > 80% of budget | Review usage patterns |
| Database connection pool | > 80% utilization | Scale or optimize queries |
| Memory usage | > 85% | Investigate leaks |
| Disk usage | > 90% | Clean up or expand |

### Health check patterns

```typescript
// Readiness check — verifies all critical dependencies
app.get('/ready', async (req, reply) => {
    const checks = {
        database: await checkDatabase(),
        redis: await checkRedis(),     // if applicable
        llm: await checkLLMProvider(), // if applicable
    };
    const healthy = Object.values(checks).every(Boolean);
    return reply.status(healthy ? 200 : 503).send({ status: healthy ? 'ready' : 'degraded', checks });
});
```

### Structured log events to monitor

| Event | Meaning |
|-------|---------|
| `http_request_completed` with `statusCode >= 500` | Server errors |
| `llm_call_failed` | LLM provider issues |
| `circuit_breaker_opened` | Dependency failure pattern |
| `auth_failure` | Authentication issues |
| `tenant_mismatch` | Potential security issue |
| `startup_failed` | Service failed to start |
