---
trigger: model_decision
scope: database_changes
---

# Database Workflow, Migrations, and Tenant Isolation

> Rules for database schema changes, Prisma migrations, and multi-tenant data safety.
> Applies whenever touching: `schema.prisma`, `migrations/`, persistence logic, or tenant-scoped data.

---

## 1. Golden Rule

**Every schema change = a migration file. No exceptions.**

If you modify `schema.prisma` (add table, change column, add relation, etc.):
- You **MUST** generate a migration file
- You **MUST NOT** rely on `db push` or manual SQL
- You **MUST NOT** leave the schema changed without a corresponding migration

---

## 2. Mandatory Workflow

### Step 1: Modify the schema
Edit the Prisma schema file (e.g., `prisma/schema.prisma`).

### Step 2: Generate migration
```bash
npx prisma migrate dev --name <descriptive_name>
```
This creates a timestamped migration folder under `prisma/migrations/` and applies it to your local database.

### Step 3: Generate Prisma client
```bash
npx prisma generate
```
Without this, TypeScript builds will fail with errors like `Property 'xyz' does not exist on type 'PrismaClient'`.

### Step 4: Verify
- [ ] New migration folder exists in `prisma/migrations/`
- [ ] Inspect the generated SQL — ensure it matches your intent
- [ ] Backend `npm run build` passes
- [ ] Backend `npm run typecheck` passes
- [ ] Affected queries still compile and work correctly

---

## 3. Production Migration Path

Production applies migrations with:
```bash
npx prisma migrate deploy
```

This command:
- Checks the `prisma/migrations/` folder
- Applies any pending migrations to the production database
- Ensures the DB schema matches code expectations

**NEVER** use `prisma db push` in production or on shared branches.

---

## 4. Tenant Isolation (Mandatory for Business Data)

### Hard rules:

1. **Business tables must carry `tenantId`**
   - Every table holding tenant-scoped business data must have a `tenantId` column

2. **Prisma queries must include tenant scoping**
   ```typescript
   // CORRECT — tenant-scoped query
   const items = await prisma.item.findMany({
       where: { tenantId: ctx.tenantId, ... }
   });

   // FORBIDDEN — unscoped query on tenant data
   const items = await prisma.item.findMany({
       where: { ... }  // Missing tenantId = cross-tenant data leak
   });
   ```

3. **Tenant identity comes from auth context, not request body**
   - For protected behavior, extract `tenantId` from the authenticated token/context
   - **Never** trust arbitrary `tenantId` from the request body for authorization
   - The dev-only `x-tenant-id` header pattern is an exception, not the normal authority model

4. **Never write queries that can accidentally read or mutate cross-tenant data**

---

## 5. Schema Evolution Strategy

### Prefer additive changes
- New optional fields (with defaults)
- New tables
- New relations with optional cardinality

### Destructive changes require extra care
If a migration is destructive or materially changes data semantics:
- **ADR required** — document the rationale and impact
- **Rollback plan** — document how to revert the migration
- **Data migration** — plan for existing data transformation
- **Consumer verification** — ensure all queries/services handle the change

---

## 6. Practical Checklist

For every database change:

1. [ ] Edit `schema.prisma`
2. [ ] Create migration: `npx prisma migrate dev --name <name>`
3. [ ] Generate client: `npx prisma generate`
4. [ ] Inspect the migration SQL
5. [ ] Check backend code that reads/writes the changed model
6. [ ] Verify tenant scoping exists on all relevant queries
7. [ ] Run backend build and tests
8. [ ] Baseline passes green

---

## 7. Troubleshooting

### Drift Detected
If `prisma migrate dev` complains about drift:
- It means the DB schema was changed outside of migrations
- You may need to reset the dev DB: `prisma migrate reset` (local only, destroys data)

### Stale Prisma Client
If TypeScript reports missing properties on Prisma models:
- Run `npx prisma generate` to regenerate the client
- Ensure the generated client version matches your schema

### Failed Migration
- Check the migration SQL for errors
- Verify the database is accessible
- Check for conflicting schema states

---

## 8. Anti-Patterns

- Editing `schema.prisma` without creating a migration
- Using `prisma db push` as a substitute for migration history
- Forgetting `prisma generate` after schema changes
- Adding a new business table without `tenantId` when the data is tenant-scoped
- Reading `tenantId` from request body for authorization-sensitive behavior
- Deploying with `prisma db push` instead of `prisma migrate deploy`
- Manual SQL as the primary schema change mechanism
- Leaving schema changes without a corresponding migration folder

---

## 9. Indexation and Query Performance

### Mandatory indexes

| Situation | Index required |
|-----------|---------------|
| Foreign key column | Index on the FK column |
| `tenantId` on business tables | Index on `tenantId` (or compound with primary query field) |
| Columns used in `WHERE` frequently | Index on the filtered column |
| Columns used in `ORDER BY` | Index if the table is large (>10k rows expected) |
| Unique business constraint | Unique index (e.g., `@@unique([tenantId, invoiceNumber])`) |

### Compound indexes for multi-tenant queries

```prisma
model Invoice {
    id        String @id @default(uuid())
    tenantId  String
    number    String
    status    String
    createdAt DateTime @default(now())

    @@index([tenantId, status])        // Queries: list by tenant + status
    @@index([tenantId, createdAt])     // Queries: list by tenant, sorted by date
    @@unique([tenantId, number])       // Business constraint: unique number per tenant
}
```

### Query patterns

```typescript
// GOOD — efficient tenant-scoped query with index
const invoices = await prisma.invoice.findMany({
    where: { tenantId, status: 'DRAFT' },
    orderBy: { createdAt: 'desc' },
    take: 50,
});

// BAD — full table scan, no tenant scoping
const invoices = await prisma.invoice.findMany({
    where: { status: 'DRAFT' },
});

// GOOD — pagination with cursor
const invoices = await prisma.invoice.findMany({
    where: { tenantId },
    cursor: lastId ? { id: lastId } : undefined,
    take: 50,
    orderBy: { createdAt: 'desc' },
});

// BAD — offset pagination on large tables
const invoices = await prisma.invoice.findMany({
    where: { tenantId },
    skip: page * 50,  // Gets slower as page increases
    take: 50,
});
```

### Rules
- Every query on a business table MUST include `tenantId` in the `where` clause
- Prefer cursor-based pagination over offset for large datasets
- Add `take` limit to every `findMany` — never return unbounded results
- Use `select` to fetch only needed fields on large models
- Add indexes BEFORE the data grows, not after performance degrades
