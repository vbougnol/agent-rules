---
trigger: model_decision
scope: feature_creation
---

# Feature Architecture and Module Creation

> Rules for creating, structuring, and wiring new features in a full-stack monorepo.
> Applies when adding any new feature or top-level module.

---

## 1. Core Principle: One Feature = One Folder = One Public Entry

Every feature is a self-contained unit:
- All feature code lives in a **single feature directory**
- Each feature exposes **one** public entry: `index.ts`
- From outside the feature folder, you may **only** import from `index.ts`

**FORBIDDEN:** importing another feature's internal files (e.g., `features/other/ui/SomeComponent`).

---

## 2. Feature Folder Structure

### Frontend Feature

```
src/features/<feature-name>/
  index.ts          # Single public entry — exports only what other features need
  README.md         # Mandatory — purpose, flow, contracts, failure modes
  ui/               # Components (pages, widgets, dialogs)
  hooks/            # Orchestration hooks
  services/         # API calls and data fetching
  types/            # Type definitions
  tests/            # Smoke and UI tests
```

### Backend Feature

```
src/features/<feature-name>/
  index.ts                    # Public entry
  <feature>.service.ts        # Business logic
  <feature>.routes.ts         # HTTP route definitions
```

For complex features with grouped sub-modules:

```
src/features/<feature-name>/
  index.ts
  read/
    index.ts
    <feature>-read.routes.ts
    <feature>-read.service.ts
  write/
    index.ts
    <feature>-write.routes.ts
    <feature>-write.service.ts
```

---

## 3. Full-Stack Feature Workflow

**Always start backend-first** when the feature needs persistence or API behavior.

### Step-by-step:

1. **Define contracts** — Zod schemas in backend `src/contracts/`
2. **Add backend services and routes** — business logic in services, HTTP handling in routes
3. **Register routes** in the backend server entry (e.g., `server.ts`) with proper prefix
4. **Verify gateway compatibility** — ensure the API gateway proxies correctly (usually automatic)
5. **Mirror contracts in frontend** — copy Zod schemas to frontend `src/contracts/`
6. **Build frontend service layer** — use environment-based API base path
7. **Build UI** — under the feature folder, using shared components
8. **Add routing** — register the route in the app router (under protected/authenticated layout)
9. **Register in module hub** — add the module to the registry for navigation visibility
10. **Add README, tests, ADR** — mandatory documentation and verification

---

## 4. Backend Routing Pattern

### Route registration
```typescript
// In server.ts — centralized registration with prefix
await server.register(async (fastify: FastifyInstance) => {
    await registerFeatureRead(fastify);
    await registerFeatureWrite(fastify);
}, { prefix: '/<feature>' });
```

### Route files use relative paths
```typescript
// In <feature>.routes.ts — no prefix here, it's set at registration
fastify.post('/boards', async (req, reply) => { ... });
// With prefix from server.ts, this becomes /<feature>/boards
```

### Gateway behavior
- Frontend calls: `/api/<feature>/<path>`
- Gateway strips `/api` and forwards to backend
- Backend receives: `/<feature>/<path>`
- **Do not** put `/api` prefix in backend routes

---

## 5. Frontend Module Wiring

### Route registration
```typescript
// In App.tsx — under authenticated layout
<Route element={<ProtectedRoute />}>
  <Route element={<MainLayout />}>
    <Route path="/apps/<feature>/*" element={<FeaturePage />} />
  </Route>
</Route>
```

### Module hub registration
```typescript
// In modules/registry.ts
{
    id: '<feature>',
    title: 'Feature Title',
    description: 'Short description.',
    icon: IconName,        // from lucide-react
    path: '/apps/<feature>',
    enabled: true
}
```

### API calls must use environment base path
```typescript
const API_BASE = import.meta.env.VITE_API_BASE_PATH || '/api';
const response = await fetch(`${API_BASE}/<feature>/endpoint`, {
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
    }
});
```

---

## 6. Shared Code Rules

Shared code is not a dumping ground. If something becomes shared, it must earn its place:
- Explicit ownership
- Tests
- Documentation
- ADR if it increases coupling between features

---

## 7. Context Window Budget

For any step or PR:
- Touch **one feature folder** at a time whenever possible
- Cross-feature changes require:
  - Explicit PLAN justification
  - ADR stating why the boundary crossing is necessary
  - Added/updated contract tests if a boundary is impacted

---

## 8. Anti-Patterns

- Importing `features/other-feature/ui/...` directly
- Calling backend URLs directly from frontend (`http://localhost:3001/...`)
- Hardcoding `/api/<feature>` without environment-based base path
- Registering only the frontend route and forgetting the module registry entry
- Changing `schema.prisma` without migration and `prisma generate`
- Building giant monolithic page components with mixed concerns
- Forgetting README in the feature folder
- Shipping features with no tests

---

## 9. Required Deliverables Per Feature

- [ ] Feature folder with proper structure
- [ ] `index.ts` as sole public entry
- [ ] `README.md` in the feature folder
- [ ] Backend contracts (Zod schemas) if full-stack
- [ ] Frontend contract mirrors if crossing HTTP boundaries
- [ ] UI smoke tests
- [ ] API/contract tests if API-dependent
- [ ] Module registry entry (if top-level module)
- [ ] Route registered in app router
- [ ] ADR for boundary/architecture decisions
- [ ] Baseline passes green
- [ ] `FEATURES.md` manifest updated (if top-level module)

---

## 10. Feature Manifest

Every project must maintain a `FEATURES.md` file at the root:

```markdown
# Features Manifest

| Feature | Status | Backend | Frontend | DB Model | API Routes | Owner |
|---------|--------|---------|----------|----------|------------|-------|
| invoices | active | src/features/invoices/ | src/features/invoices/ | Invoice | /api/invoices/* | team-billing |
| auth | active | src/features/auth/ | src/features/auth/ | User, Session | /api/login, /api/me | team-platform |
```

This file is the **entry point for any AI agent** starting a maintenance session. It provides a complete map of the project in seconds.

Update it every time a new feature is added or an existing feature changes status.

---

## 11. Naming Convention Alignment

All feature names must follow the conventions in `10-naming-conventions.md`. Quick reference:

| Element | Derives from feature name `<feature>` |
|---------|---------------------------------------|
| Backend folder | `src/features/<feature>/` |
| Backend service | `<feature>.service.ts` |
| Backend routes | `<feature>.routes.ts` |
| Contract | `src/contracts/<feature>.contract.ts` |
| Frontend folder | `src/features/<feature>/` |
| Frontend page | `<Feature>Page.tsx` |
| Frontend API hook | `use<Feature>Api.ts` |
| API route | `/api/<feature>/*` |
| DB model | `<Feature>` (PascalCase, singular) |
| Tests | `<feature>.service.test.ts`, `<Feature>Page.test.tsx` |

An AI agent must be able to navigate the entire feature by knowing only its name.
