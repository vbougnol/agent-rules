# Contributing to Agent Rules

## Before updating a rule

1. Understand the priority hierarchy: Correctness > Security > Operability > Maintainability > Performance > Convenience
2. Read the Engineering Constitution (`00-engineering-constitution.md`)
3. Check if your change affects multiple files (likely needs a coordinated update)

## How to propose changes

1. Create a branch: `feat/improve-<aspect>` or `fix/<specific-rule>`
2. Update the relevant rule file(s)
3. If a new file, add it to the README.md index
4. If cross-referenced, update all related files
5. Submit a PR with a clear explanation of why the change improves maintainability or AI compatibility

## Review criteria

- Does the change follow the priority hierarchy?
- Is it clear for both AI agents and humans?
- Does it improve production-readiness or maintainability?
- Does it introduce unnecessary complexity?
