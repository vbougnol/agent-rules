---
trigger: model_decision
scope: api_development
---

# API Contracts, Gateway, and Communication

> Rules for frontend-backend communication, API gateway patterns, contract-first development,
> and multi-tenant request handling. Applies whenever touching API calls, routes, contracts, or gateway.

---

## 1. Architecture: Frontend > Gateway > Backend

```
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│   FRONTEND   │───────>│  API GATEWAY │───────>│   BACKEND    │
│  (React/Vite)│  HTTP  │  (Proxy)     │  HTTP  │  (Fastify)   │
│              │<───────│              │<───────│              │
└──────────────┘        └──────────────┘        └──────────────┘
     │                        │                        │
  Contracts              Proxy Layer              Contracts
  (Zod Mirror)          Security/CORS            (Zod Source)
```

---

## 2. Non-Negotiable Rules

### Rule #1: NEVER call backend directly from frontend

```typescript
// FORBIDDEN
fetch('http://localhost:3001/chat', { ... })

// MANDATORY
const apiBase = import.meta.env.VITE_API_BASE_PATH || '/api';
fetch(`${apiBase}/chat`, { ... })
```

**Why?** Direct calls bypass CORS, rate-limiting, security headers, and tracing.

### Rule #2: Always validate with Zod contracts

```typescript
// FORBIDDEN — implicit types or any
const data: any = await res.json();

// MANDATORY — runtime validation
import { ResponseSchema } from '@/contracts/feature.contract';
const rawData = await res.json();
const data = ResponseSchema.parse(rawData);
```

### Rule #3: Standardized headers

**Required for authenticated requests:**
- `Authorization: Bearer <token>`
- `Content-Type: application/json` (for POST/PUT/PATCH)
- `x-tenant-id: <tenant-id>` (if multi-tenant)

**Automatic (injected by gateway):**
- `x-request-id` — correlation ID for tracing

---

## 3. Contract Structure

### Backend = Source of Truth

```
src/backend/src/contracts/
├── auth.contract.ts
├── feature-a.contract.ts
├── feature-b.contract.ts
└── ...
```

### Frontend = Mirror

```
src/frontend/src/contracts/
├── auth.contract.ts        ← Exact copy from backend
├── feature-a.contract.ts   ← Exact copy from backend
└── ...
```

### Contract rules:
- Backend contracts are **always** the source of truth
- Frontend mirrors those contracts when crossing HTTP boundaries
- Infer TypeScript types from Zod schemas — do not duplicate type shapes manually
- Changes must be **additive** unless an ADR explicitly authorizes breaking changes
- No silent breaking contract changes

---

## 4. Gateway Behavior

The gateway serves as a reverse proxy with security enforcement:

```
Frontend Request:   GET /api/feature/items
       ↓
Gateway Receives:   GET /api/feature/items
       ↓
Gateway Rewrites:   GET /feature/items  (strips /api)
       ↓
Backend Receives:   GET /feature/items
```

**Rules:**
- Gateway strips `/api` prefix before forwarding
- Do not design backend routes assuming they're under `/api`
- Do not modify gateway code for standard `/api/*` features
- Gateway injects `x-request-id` — never break this propagation

### Gateway security headers (do not remove)
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: no-referrer`

---

## 5. Authentication Flow

```typescript
// 1. Login (unauthenticated)
const apiBase = import.meta.env.VITE_API_BASE_PATH || '/api';
const response = await fetch(`${apiBase}/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
});

// 2. Parse + validate response
const { token } = LoginResponseSchema.parse(await response.json());

// 3. Store token
localStorage.setItem('auth_token', token);

// 4. Use token for subsequent calls
const res = await fetch(`${apiBase}/resource`, {
    headers: { 'Authorization': `Bearer ${token}` }
});
```

### 401 Handling is MANDATORY

```typescript
if (response.status === 401) {
    logout();  // Clear token + redirect to login
    throw new Error('Session expired');
}
```

**NEVER** swallow 401s and leave the UI in a fake logged-in state.

---

## 6. Communication Patterns

### Pattern: GET request
```typescript
const fetchData = async (token: string) => {
    const apiBase = import.meta.env.VITE_API_BASE_PATH || '/api';
    const res = await fetch(`${apiBase}/resource`, {
        headers: { 'Authorization': `Bearer ${token}` }
    });
    if (res.status === 401) { logout(); throw new Error('Unauthorized'); }
    if (!res.ok) { throw new Error(`HTTP ${res.status}`); }
    return ResponseSchema.parse(await res.json());
};
```

### Pattern: POST request
```typescript
const createItem = async (token: string, payload: CreateRequest) => {
    const apiBase = import.meta.env.VITE_API_BASE_PATH || '/api';
    const res = await fetch(`${apiBase}/resource`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify(payload)
    });
    if (res.status === 401) { logout(); throw new Error('Unauthorized'); }
    if (!res.ok) { throw new Error(`HTTP ${res.status}`); }
    return CreatedResponseSchema.parse(await res.json());
};
```

### Pattern: File upload (FormData)
```typescript
const uploadFile = async (token: string, file: File) => {
    const apiBase = import.meta.env.VITE_API_BASE_PATH || '/api';
    const formData = new FormData();
    formData.append('file', file);
    const res = await fetch(`${apiBase}/upload`, {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${token}` },
        // Do NOT set Content-Type — fetch sets it automatically for FormData
        body: formData
    });
    if (!res.ok) { throw new Error('Upload failed'); }
    return res.json();
};
```

### Pattern: Centralized API hook
```typescript
export const useFeatureApi = () => {
    const { token, logout } = useAuth();
    const API_BASE = import.meta.env.VITE_API_BASE_PATH || '/api';

    const _fetch = async <T>(path: string, options: RequestInit = {}): Promise<T> => {
        const res = await fetch(`${API_BASE}${path}`, {
            ...options,
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${token}`,
                ...options.headers
            }
        });
        if (res.status === 401) { logout(); throw new Error('Unauthorized'); }
        if (!res.ok) { throw new Error((await res.json()).message || 'Request failed'); }
        return res.json();
    };

    return {
        getItems: () => _fetch<Item[]>('/items'),
        createItem: (data: CreateRequest) => _fetch<Item>('/items', {
            method: 'POST',
            body: JSON.stringify(data)
        }),
    };
};
```

---

## 7. Multi-Tenant Safety

- Tenant-scoped endpoints must enforce `tenantId` from **authenticated context/token**
- **Never** trust client-supplied `tenantId` in the request body for authorization decisions
- Scope all reads and writes by tenant
- Admin endpoints must come from auth context, not body

---

## 8. HTTP Error Handling

| Code | Meaning | Frontend Action |
|------|---------|-----------------|
| `200` | OK | Process response |
| `201` | Created | Resource created |
| `400` | Bad Request | Show validation error |
| `401` | Unauthorized | **Logout + redirect to login** |
| `403` | Forbidden | Show "Access denied" |
| `404` | Not Found | Show "Resource not found" |
| `429` | Too Many Requests | Show "Slow down" |
| `500` | Server Error | Show "Server error, try again" |

---

## 9. Standard Error Response Format

All API error responses MUST follow this shape (see `11-error-handling.md` for full details):

```typescript
{
    error: true,
    code: "INVOICE_NOT_FOUND",     // Machine-readable, UPPER_SNAKE_CASE
    message: "Invoice not found",   // Human-readable
    statusCode: 404,
    requestId: "req-abc-123",       // Correlation ID
    retryable: false,               // Can the client retry?
    details?: { ... }               // Optional: validation errors, context
}
```

**Rules:**
- Backend MUST use the centralized error handler — never craft error responses ad hoc
- Frontend MUST use the standardized error hook — never handle errors inconsistently
- Error `code` follows convention: `<DOMAIN>_<DESCRIPTION>` in `UPPER_SNAKE_CASE`
- The `retryable` field tells the frontend whether to offer a retry button

---

## 10. Retry and Resilience

### Retry policy for frontend API calls

| HTTP Status | Retryable? | Strategy |
|-------------|-----------|----------|
| `400` | No | Fix input |
| `401` | No | Re-authenticate |
| `403` | No | No retry |
| `404` | No | No retry |
| `429` | Yes | Backoff + respect `Retry-After` header |
| `500` | Yes | 2-3 retries with exponential backoff |
| `502/503` | Yes | Exponential backoff |
| `504` | Yes | Retry with increased timeout |
| Network error | Yes | Exponential backoff |

### Default retry config

```typescript
{ maxRetries: 3, baseDelayMs: 1000, maxDelayMs: 30000, backoffMultiplier: 2 }
```

Always add jitter (random 0-200ms) to avoid thundering herd.

---

## 11. Anti-Patterns

- `fetch('http://localhost:3001/...')` in frontend code
- Backend routes relying on implicit, undocumented payload shapes
- Contract drift between backend and frontend
- Body-provided `tenantId` treated as authorization truth
- Repeated ad hoc `fetch` code across multiple components
- Ignoring 401 errors (leaving UI in fake logged-in state)
- Using `any` type for API responses
- Hardcoding API URLs
- Inconsistent error response formats between features
- No retry logic for transient failures (503, network errors)
- Retry on non-retryable errors (400, 401, 403)
