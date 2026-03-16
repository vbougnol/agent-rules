---
trigger: always_on
scope: all_projects
---

# Engineering Constitution (Non-Negotiable)

> These rules apply to **every** code change in **every** project.
> They are non-negotiable and must be followed by all AI agents and developers.

---

## 0. Priority Hierarchy

When trade-offs are necessary, resolve in this order:

**Correctness > Security > Operability > Maintainability > Performance > Convenience**

---

## 1. Prime Directive

Ship code that is:
- **Correct** — does what it claims, handles edge cases
- **Secure** — zero-trust by default, secrets never in code
- **Operable** — observable, diagnosable, safe in production
- **Auditable** — decisions traceable, changes reviewable, behavior explainable
- **Maintainable** — low coupling, high cohesion, predictable structure
- **Debuggable** — structured logs, clear errors, reproducible paths
- **Human-readable** — explicit names, small units, documented intent
- **AI-maintainable** — stable patterns, explicit contracts, no "clever" shortcuts
- **Extensible** — new features plug in with minimal ripple effects

---

## 2. Operating Protocol: PLAN > ACT > VERIFY > DOCUMENT

Follow this cycle for **every** change.

### 2.1 PLAN (before touching code)

Before writing any code, produce:
- **Goal and non-goals** — what you're solving and what's out of scope
- **Constraints** — stack, existing patterns, contracts that must not change
- **Boundaries touched** — API, DB, UI, feature boundaries
- **Failure modes** — invalid inputs, timeouts, overload, concurrency
- **Minimal change set** — smallest set of edits that achieves the goal
- **Test strategy** — what to run, where, and why
- **Rollback plan** — how to revert safely

No code generation in PLAN mode unless explicitly requested for a spike/prototype.

### 2.2 ACT (atomic changes)

- One concern per change group
- Prefer **extraction** and **composition** over rewriting
- No "big bang" refactors unless explicitly requested and justified
- Small, reviewable steps

### 2.3 VERIFY (mandatory)

- Lint and typecheck pass
- Unit tests pass
- Contract/integration tests when boundaries are touched
- Negative path testing (invalid inputs, empty states, timeouts)
- If contracts changed: verify consumers or keep changes strictly additive
- Run the project's baseline/CI gate

### 2.4 DOCUMENT (mandatory)

For every change:
- Write/update an **ADR** (Architecture Decision Record) for non-trivial decisions
- Update README/runbook if operational behavior changes
- Use **Conventional Commits**: `feat:`, `fix:`, `chore:`, `refactor:`, `test:`, `docs:`

---

## 3. Code Quality Rules

### 3.1 TypeScript strict mode
- **Never** use `any` unless: documented why, contained at a boundary, compensated by tests
- Descriptive names, explicit control flow, no cleverness
- Small units, prefer pure functions when practical

### 3.2 Defensive programming at boundaries
At external boundaries (HTTP, DB, queue, files, user input):
- Validate inputs with schemas
- Normalize data
- Handle invalid states explicitly

### 3.3 No TODO/FIXME in production paths
Unless: linked to a tracked issue AND not in a critical path AND not a runtime risk.

---

## 4. Architecture Principles

### 4.1 Incremental over big-bang
- Prefer "seams" and safe extraction points
- Preserve external behavior (contracts, status codes, error shapes) unless explicitly approved
- Domain/use-case logic should be separated from IO where it pays off

### 4.2 Feature isolation
- One feature = one folder, one public entry point (`index.ts`)
- No cross-feature deep imports — only import from `features/<name>/index.ts`
- Shared code must earn its place: explicit ownership, tests, documentation

### 4.3 Layered architecture
- **Routes/Controllers** — HTTP handling only, no business logic
- **Services** — business logic, orchestration
- **Data Access** — persistence, external APIs
- Keep layers clean: no upward dependencies

---

## 5. Contract-First Development

### 5.1 Contracts must be explicit
Every boundary must have:
- Typed DTOs / schemas (Zod recommended)
- Stable response shapes
- Explicit error behavior

### 5.2 Compatibility rule
- **Do not change public API contracts** unless explicitly authorized
- Prefer **additive** changes: new fields optional, no renames/removals without versioning plan
- Breaking changes require ADR + proof it won't break consumers
- No silent breaking contract changes

---

## 6. Structured Logging (mandatory)

- JSON or key/value structured logs
- Must include: request/correlation ID, operation name, duration, status code, error type
- Use stable event keys: `event="http_request_completed"`, etc.
- **Never** use raw `console.log` in production
- **Never** log secrets (passwords, tokens, API keys)

---

## 7. Health & Readiness

If service-like:
- Liveness endpoint (`/health` or `/healthcheck`)
- Readiness endpoint (`/ready`) checks critical dependencies
- Timeouts everywhere (HTTP clients, DB queries)

---

## 8. Stop Conditions

Stop and request clarification if:
- Ambiguity impacts correctness, security, or contracts
- Auth rules are unclear
- A new dependency is suggested without human approval
- Breaking a public contract is implied without versioning guidance

---

## 9. Definition of Done

A change is **done** only if:
- Code compiles and typechecks
- Tests exist and pass (including edge cases)
- Observability is in place (logs/errors/health)
- ADR updated for non-trivial decisions
- Feature boundary respected
- Rollback is possible and documented
- Project baseline is green

---

## 10. AI-Maintainability Patterns

Code must be designed for maintenance by AI agents without human oversight.

### 10.1 Predictive navigation
- File names, function names, routes, and models must follow strict naming conventions (see `10-naming-conventions.md`)
- An AI agent must be able to **predict** all file paths from the feature name alone — no searching needed
- Every feature folder must have a `README.md` and an `index.ts` public entry

### 10.2 Self-documenting code
- Structure tells the story: group queries vs commands, organize by domain concern
- Function names = verb + noun (`createInvoice`, not `process`)
- Boolean names = questions (`isActive`, `hasPermission`, `canDelete`)
- No abbreviations that require domain knowledge to decode
- Constants express intent (`MAX_RETRY_ATTEMPTS`, not `NUM_3`)

### 10.3 Zero magic
- No implicit behavior: every side effect is explicit and documented
- No "clever" code: readable beats concise
- No hidden state: state flows through explicit parameters, not global mutations
- No naming inconsistency between layers (if the DB calls it `Invoice`, the API calls it `Invoice`, the UI calls it `Invoice`)

### 10.4 Standardized error shapes
- Every API uses the same error format (see `11-error-handling.md`)
- Error codes are machine-readable (`INVOICE_NOT_FOUND`) and follow `<DOMAIN>_<DESCRIPTION>` pattern
- AI agents can write error handling generically because the shape is always the same

### 10.5 Feature manifest
- Maintain a `FEATURES.md` at the project root listing all features, their status, paths, models, and routes (see `13-ai-maintenance-protocol.md`)
- This allows any AI agent to understand the entire project in seconds

### 10.6 Idempotent operations
- All mutations should be designed for safe retry (see `11-error-handling.md`)
- If an AI agent accidentally re-runs an operation, the system must be safe

---

## 11. Output Contract (Agent Responses)

Unless requested otherwise, agent output includes:
1. Assumptions and risks
2. Plan
3. Changeset (grouped by file, diff preferred)
4. Commands to verify (lint/test/run)
5. Rollback steps
6. ADR path + summary (if applicable)

No unrelated refactors. No cosmetic churn unless requested.
