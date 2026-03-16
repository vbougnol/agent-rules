---
mode: CREATE_FEATURE
scope: strict_feature_creation
---

# Feature Creation Prompt (Strict Mode)

> Use this prompt template when creating a new feature from scratch.
> It enforces minimal blast radius, boundary safety, and mandatory verification.

---

## Context

- Follow the Engineering Constitution (`00-engineering-constitution.md`)
- Follow Feature Architecture rules (`02-feature-architecture.md`)
- Follow UI/UX Design System (`05-ui-ux-design-system.md`) if the feature has UI
- Follow API Contracts rules (`03-api-contracts-gateway.md`) if the feature crosses HTTP boundaries
- Follow Database Workflow (`04-database-workflow.md`) if the feature touches persistence

---

## Non-Negotiable Rules

1. **Scope control:** You may create/modify files ONLY in:
   - `src/features/<FEATURE_NAME>/**`
   - `docs/adr/**` (only if boundary/decision is introduced)

2. **Single external edit:** The ONLY allowed edit outside the feature folder is:
   - Registering the feature entry in the app's module hub / router / navigation (ONE place)
   - If more edits are needed, STOP and explain why

3. **Contract preservation:** Do NOT change existing API contracts, schemas, or backend behavior

4. **Dependency freeze:** Do NOT add new dependencies unless explicitly approved by the human

5. **Tests mandatory:** At least a UI smoke test + contract usage test (if applicable)

6. **Baseline must pass:** After implementation, `npm run baseline` must be green

---

## Feature Specification Template

Fill this in before starting:

```
- Feature name (kebab-case): <FEATURE_NAME>
- User goal: <what the user achieves with this feature>
- Type: [frontend-only | full-stack]

- UI:
  - Screen(s): <list of screens>
  - Main components: <key components to build>

- Data:
  - API calls used (existing endpoints only): <list>
  - Contract(s) consumed (existing Zod schemas): <list>
  - New contracts needed: <list, if any>

- States to handle:
  - [ ] Loading
  - [ ] Success
  - [ ] Empty
  - [ ] Error (401 / 403 / 429 / timeout / 5xx)

- Failure modes:
  - <describe what can go wrong and how to handle it>
```

---

## Tasks

### A) PLAN (no code yet)
- Propose folder tree under `src/features/<FEATURE_NAME>/`
- List exact files to create
- List the ONE external file to edit for registration (path + what change)
- Define tests to add and what they prove
- Identify contracts needed (existing vs new)

### B) ACT (implement)
- Create the feature folder and files:
  - `index.ts` (public exports)
  - `README.md` (intent, flow, contracts, failure modes, smoke checks)
  - `ui/`, `hooks/`, `services/`, `types/`, `tests/` as needed
- Implement the feature using existing contracts (or create new ones backend-first)
- Add tests
- Register feature in the allowed external place

### C) VERIFY
- Run `npm run baseline` and report PASS
- Verify routes work through gateway (frontend > gateway > backend)
- Verify module appears in navigation (if top-level module)

### D) DOCUMENT
- If any boundary/architectural decision was made, write an ADR in `docs/adr/`
- Feature README is complete and accurate

---

## Output Format

```
## Plan
<folder tree, files list, registration point>

## Changeset
<grouped by file, diff preferred>

## Commands to Verify
<lint, typecheck, test, build, baseline>

## Rollback Steps
<how to revert if something goes wrong>

## ADR
<path and 2-line summary, if applicable>
```
