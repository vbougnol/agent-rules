---
trigger: model_decision
scope: verification
---

# Quality Gates, Testing, and Documentation

> Rules for testing, baseline verification, and documentation requirements.
> Applies to any non-trivial implementation touching tests, boundaries, or public behavior.

---

## 1. Baseline Must Stay Green

The project baseline is the primary quality gate:

```bash
npm run baseline
```

This typically runs across all workspaces:
- **Lint** — code style and static analysis
- **Typecheck** — TypeScript strict mode compilation
- **Test** — unit and integration tests
- **Build** — production build verification
- **Boundary check** — feature isolation enforcement (if configured)

### Rules:
- Baseline must pass **before** and **after** every change
- If baseline is already broken before the change: either fix it or explicitly document that it blocks safe completion
- **Never** declare work "done" with a red baseline

---

## 2. Testing Pyramid

### Required test types by context:

| Context | Required Tests |
|---------|---------------|
| Core business logic | Unit tests |
| Adapters / integrations | Integration tests (when needed) |
| API boundaries | Contract tests (Zod validation) |
| Critical user paths | Minimal E2E tests |
| UI features | Smoke tests |
| Every feature | At least one negative path test |

### Edge cases are first-class:
Every feature change must include tests for:
- **Invalid input** — malformed data, wrong types, missing fields
- **Empty state** — no data, null values
- **Timeout/overload** — when applicable
- **Idempotency** — when applicable (repeated calls produce same result)

### Test quality rules:
- **No flaky tests** — control randomness/time (inject clock, seed RNG)
- Tests prove **behavior**, not implementation details
- Tests must actually cover the touched behavior (not just exist)

---

## 3. Documentation Requirements

### README per feature (mandatory)

Every feature folder must have a `README.md` explaining:
- Purpose and user value
- Critical flow description
- Contracts/endpoints involved
- Failure modes and edge cases
- How to verify/test

### Architecture Decision Records (ADRs)

Create or update an ADR when the change:
- Introduces or modifies a boundary
- Changes architecture or routing strategy
- Changes a contract non-additively
- Adds a new dependency
- Changes baseline/test strategy
- Changes operational behavior or deployment assumptions

ADR structure:
```markdown
# ADR: <Title>

## Context
What situation led to this decision?

## Decision
What did we decide?

## Alternatives Considered
What other options were evaluated? (brief)

## Consequences
What are the implications of this decision?
```

### Operational documentation

If the change affects startup, deploy/rollback, smoke checks, or critical user flows:
- Update the relevant docs/runbooks
- Do not leave changed behavior undocumented

---

## 4. Conventional Commits

Use standardized commit messages:

| Prefix | Usage |
|--------|-------|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `chore:` | Maintenance, dependencies |
| `refactor:` | Code restructuring without behavior change |
| `test:` | Adding or updating tests |
| `docs:` | Documentation changes |

---

## 5. Verification Checklist

Before closing any implementation:

- [ ] Run root `npm run baseline` (or targeted workspace commands)
- [ ] New or changed tests cover the touched behavior
- [ ] Feature README exists and is still accurate
- [ ] ADR need was evaluated (not ignored)
- [ ] No silent breaking changes to public behavior
- [ ] Rollback plan is documented for non-trivial changes
- [ ] Observability in place (logs, error handling, health endpoints)

---

## 6. Definition of Done

A feature is **done** only when:

- [ ] Code compiles and typechecks
- [ ] Tests exist and pass (including edge cases)
- [ ] Observability is in place (logs/errors/health)
- [ ] ADR updated for non-trivial decisions
- [ ] Feature boundary respected
- [ ] Rollback is possible and documented
- [ ] Baseline is green
- [ ] README exists in the feature folder

---

## 7. Anti-Patterns

- Shipping feature work with no tests
- Updating routes/contracts without verifying consumers
- Treating documentation as optional after changing public behavior
- Declaring work "done" when baseline is knowingly red
- Adding a feature folder without a README
- Writing tests that don't actually test the changed behavior
- Skipping negative path testing for critical flows
- No ADR for boundary-changing decisions
