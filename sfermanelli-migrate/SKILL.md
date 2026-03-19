---
name: sfermanelli-migrate
description: Plan and execute migrations — framework upgrades, dependency updates, database schema changes, and breaking change adoption. Use this skill when the user asks to upgrade a dependency, migrate to a new version, update a framework, handle breaking changes, run a database migration, or says things like "upgrade to v5", "migrate from X to Y", "update dependencies", "handle breaking changes", "schema migration". Distinct from refactor (which improves structure) — this changes the underlying platform or dependencies.
---

# Migrate

Plan and execute migrations safely. Migrations are high-risk operations — a bad migration can take down production, corrupt data, or leave the codebase in an inconsistent state. Move in small, verifiable steps with a rollback plan for each.

## Golden Rule

**Every migration step must be independently deployable and reversible.** If step 3 fails, steps 1 and 2 should still work in production. Never put the system in a state where you must go forward to survive.

---

## 1. Migration Process

### The 5-Step Migration

1. **Assess** — What's changing? What breaks? What's the blast radius?
2. **Plan** — Break into the smallest possible steps. Define rollback for each step.
3. **Prepare** — Set up compatibility layers, feature flags, dual-write paths
4. **Execute** — Apply changes incrementally. Verify each step before proceeding.
5. **Clean up** — Remove compatibility code, old dependencies, feature flags

### Assessment Checklist

```markdown
## Migration Assessment: [from] → [to]

### Breaking Changes

- [ ] API changes — list every breaking change from changelog/migration guide
- [ ] Type changes — new generics, removed types, changed signatures
- [ ] Behavioral changes — same API but different behavior (most dangerous)
- [ ] Dependency conflicts — peer deps, transitive deps that need updating too
- [ ] Configuration changes — new required env vars, changed defaults
- [ ] Database changes — schema modifications, new constraints

### Blast Radius

- [ ] Files affected: X
- [ ] Tests affected: X
- [ ] Downstream consumers: [list services/packages that depend on this]
- [ ] Database changes required: yes/no
- [ ] Environment variable changes: [list]
- [ ] CI/CD changes required: yes/no

### Risk Level

- [ ] Low — API renames, deprecation cleanup, type adjustments
- [ ] Medium — behavioral changes, new patterns required, new peer deps
- [ ] High — database changes, multiple services affected, auth changes
- [ ] Critical — data migration, irreversible schema changes, external API contracts

### Rollback Plan

- Step 1 rollback: [how]
- Step 2 rollback: [how]
- Point of no return: [which step, if any — and why it's irreversible]
```

---

## 2. Database Schema Migrations

### Prisma Migrations

```typescript
// Prisma provides a declarative migration workflow
// schema.prisma defines the desired state, Prisma generates SQL

// Step 1: Modify schema.prisma
model Order {
  id          String      @id @default(uuid())
  status      OrderStatus @default(DRAFT)
  total       Int         // cents
  // NEW: adding a column
  priority    Int         @default(0)
  customerId  String
  customer    Customer    @relation(fields: [customerId], references: [id])
  items       OrderItem[]
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@index([customerId])
  @@index([status, createdAt])  // Composite index for common queries
}

// Step 2: Generate migration
// npx prisma migrate dev --name add_order_priority

// Step 3: Review the generated SQL before applying
// prisma/migrations/20240315_add_order_priority/migration.sql
// ALTER TABLE "Order" ADD COLUMN "priority" INTEGER NOT NULL DEFAULT 0;

// Step 4: Apply to production
// npx prisma migrate deploy
```

### Expand-Contract Pattern

The safest way to change database schemas in production. Never make a breaking change in one step.

```
Timeline:

  Deploy 1 (Expand)     Deploy 2 (Migrate)    Deploy 3 (Contract)
  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
  │ Add new column   │   │ Backfill data    │   │ Drop old column  │
  │ Keep old column  │   │ Switch reads     │   │ Remove compat    │
  │ Write to both    │   │ Write to new     │   │ code             │
  └─────────────────┘   └─────────────────┘   └─────────────────┘
        Can rollback ✓       Can rollback ✓       Point of no return ✗
```

#### Example: Renaming a Column

```typescript
// WRONG — breaks everything between deploy and code update
// ALTER TABLE users RENAME COLUMN name TO full_name;

// RIGHT — expand-contract in 3 deploys

// Deploy 1: EXPAND — add new column, write to both
// migration: add_full_name_column
await prisma.$executeRaw`ALTER TABLE users ADD COLUMN full_name TEXT`
await prisma.$executeRaw`UPDATE users SET full_name = name WHERE full_name IS NULL`

// Code: write to both columns
class PrismaUserRepository implements UserRepository {
  async save(user: User): Promise<void> {
    await this.prisma.user.update({
      where: { id: user.id },
      data: {
        name: user.fullName, // old column — still active
        full_name: user.fullName, // new column — being populated
      },
    })
  }

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } })
    // Read from new column, fall back to old
    return data ? this.toDomain(data, data.full_name ?? data.name) : null
  }
}

// Deploy 2: MIGRATE — switch reads to new column, stop writing to old
// Backfill any remaining rows
await prisma.$executeRaw`UPDATE users SET full_name = name WHERE full_name IS NULL`

class PrismaUserRepository implements UserRepository {
  async save(user: User): Promise<void> {
    await this.prisma.user.update({
      where: { id: user.id },
      data: { full_name: user.fullName }, // only new column
    })
  }

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } })
    return data ? this.toDomain(data, data.full_name!) : null
  }
}

// Deploy 3: CONTRACT — drop old column
// migration: drop_name_column
await prisma.$executeRaw`ALTER TABLE users DROP COLUMN name`
// Update schema.prisma to remove the old field
```

#### Example: Changing a Column Type

```typescript
// Changing order.total from Float to Int (cents)

// Deploy 1: Add new column
// migration: add_total_cents
await prisma.$executeRaw`ALTER TABLE orders ADD COLUMN total_cents INTEGER`
await prisma.$executeRaw`UPDATE orders SET total_cents = ROUND(total * 100)`

// Code: write to both, read from new
async save(order: Order): Promise<void> {
  await this.prisma.order.update({
    where: { id: order.id.value },
    data: {
      total: order.total.cents / 100,  // old column (float)
      total_cents: order.total.cents     // new column (int)
    }
  })
}

// Deploy 2: Stop writing to old, drop it
// Deploy 3: Rename total_cents → total in schema if desired
```

#### Example: Adding a Required Relation

```typescript
// Adding a required categoryId to Product

// WRONG — adding NOT NULL column with no default breaks existing rows
// ALTER TABLE products ADD COLUMN category_id UUID NOT NULL REFERENCES categories(id);

// RIGHT — three-step migration

// Step 1: Add nullable column
// migration: add_category_to_products
model Product {
  categoryId String?  // Nullable first
  category   Category? @relation(fields: [categoryId], references: [id])
}

// Step 2: Backfill data
async function backfillProductCategories(): Promise<void> {
  const defaultCategory = await prisma.category.findFirst({ where: { slug: 'uncategorized' } })

  let cursor: string | undefined
  while (true) {
    const batch = await prisma.product.findMany({
      where: { categoryId: null },
      take: 500,
      ...(cursor && { skip: 1, cursor: { id: cursor } }),
      orderBy: { id: 'asc' }
    })

    if (batch.length === 0) break

    await prisma.product.updateMany({
      where: { id: { in: batch.map(p => p.id) } },
      data: { categoryId: defaultCategory.id }
    })

    cursor = batch[batch.length - 1].id
    console.log(`Backfilled ${batch.length} products, cursor: ${cursor}`)
  }
}

// Step 3: Make NOT NULL after all rows have values
// migration: make_category_required
await prisma.$executeRaw`ALTER TABLE products ALTER COLUMN category_id SET NOT NULL`
// Update schema.prisma:
model Product {
  categoryId String    // Now required
  category   Category  @relation(fields: [categoryId], references: [id])
}
```

### Safe Migration Rules

| Do                                          | Don't                                            |
| ------------------------------------------- | ------------------------------------------------ |
| Add nullable columns                        | Add NOT NULL without defaults                    |
| Add columns with defaults                   | Rename columns directly                          |
| Create new tables                           | Drop columns that are still read                 |
| Add indexes concurrently                    | Add indexes on large tables without CONCURRENTLY |
| Backfill in batches (500-1000 rows)         | Backfill entire table in one transaction         |
| Deploy code before migration (reads)        | Deploy migration before code is ready            |
| Test migration against production-size data | Test only against empty/small databases          |
| Generate and review SQL before applying     | Blindly run `migrate deploy`                     |
| Keep migrations small and focused           | Bundle multiple schema changes in one migration  |

### Index Management

```typescript
// Adding indexes to large tables — do it concurrently to avoid locking

// Prisma doesn't support CONCURRENTLY — use raw SQL for large tables
// migration: add_index_orders_customer_id
await prisma.$executeRaw`
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_id
  ON orders (customer_id)
`

// For composite indexes — think about query patterns
// This index supports: WHERE status = X ORDER BY created_at
await prisma.$executeRaw`
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_status_created
  ON orders (status, created_at DESC)
`

// Partial indexes — smaller, faster, for common queries
await prisma.$executeRaw`
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_pending
  ON orders (created_at DESC) WHERE status = 'PENDING'
`

// Removing unused indexes — they slow down writes for no benefit
// Check pg_stat_user_indexes for index usage statistics
// SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

---

## 3. Framework Upgrades

### TypeScript Version Upgrades

```typescript
// Strategy: fix in dependency order
// shared/ → domain/ → application/ → infrastructure/ → tests/

// 1. Update tsconfig.json — enable stricter checks gradually
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,  // TS 4.1+ — arrays can be undefined
    "exactOptionalPropertyTypes": true  // TS 4.4+ — undefined ≠ optional
  }
}

// 2. Common TS upgrade issues and fixes

// Issue: noUncheckedIndexedAccess makes array access T | undefined
const items = ['a', 'b', 'c']
const first = items[0]  // string | undefined now
// Fix: null check or non-null assertion (if you know the index is valid)
if (first) { console.log(first) }

// Issue: exactOptionalPropertyTypes
interface Config {
  port?: number  // This means "port is optional"
}
const config: Config = { port: undefined }  // Error! Can't explicitly set undefined
// Fix: use union if you need to explicitly pass undefined
interface Config {
  port?: number | undefined
}

// 3. Adopt new features ONLY AFTER all errors are fixed
// TS 5.0+: satisfies operator
const config = { port: 3000, host: 'localhost' } satisfies Config
// TS 5.0+: const type parameters
function first<const T extends readonly unknown[]>(arr: T): T[0] { return arr[0] }
```

### Next.js Major Upgrades

```markdown
## Upgrade Strategy: Next.js [old] → [new]

### Step 1: Read the migration guide completely

- https://nextjs.org/docs/upgrading
- Run the codemod: `npx @next/codemod@latest [version] .`

### Step 2: Update dependencies together

npm install next@latest react@latest react-dom@latest
npm install -D @types/react@latest @types/react-dom@latest

### Step 3: Fix in priority order

1. **Compiler errors** — won't build without fixing these
2. **Runtime errors** — build succeeds but pages break
3. **Deprecation warnings** — working but using old patterns

### Step 4: Adopt new patterns incrementally

- Don't rewrite working pages — migrate as you touch them
- New features go in new patterns, old features stay until touched
- App Router and Pages Router can coexist during migration

### Common Next.js migration issues:

- Router changes (useRouter from next/navigation vs next/router)
- Data fetching (getServerSideProps → server components)
- Image component API changes
- Middleware API changes
- Font loading changes
```

### NestJS Major Upgrades

```markdown
## NestJS Upgrade Strategy

### Step 1: Update core packages together

npm install @nestjs/core@latest @nestjs/common@latest @nestjs/platform-express@latest
npm install -D @nestjs/cli@latest @nestjs/testing@latest

### Step 2: Update companion packages

- @nestjs/swagger
- @nestjs/config
- @nestjs/typeorm or @nestjs/prisma
- @nestjs/jwt, @nestjs/passport

### Step 3: Common breaking changes

- Decorator API changes
- Module metadata changes
- Middleware signature changes
- Guard/Interceptor/Pipe interface changes

### Step 4: Run tests at each step

npm run test
npm run test:e2e
npm run build
```

### Angular Major Upgrades

```markdown
## Angular Upgrade Strategy

### Step 1: Use the official update tool

ng update @angular/core @angular/cli

### Step 2: Follow the interactive guide

- https://angular.dev/update-guide
- Select your current and target versions
- Follow every step — don't skip

### Step 3: Update third-party packages

ng update @angular/material
npm update [other-angular-packages]

### Step 4: Common breaking changes per version

- Standalone components (Angular 14+)
- Signal-based reactivity (Angular 16+)
- New control flow syntax @if/@for (Angular 17+)
- Zoneless change detection (Angular 18+)
- Module resolution changes

### Step 5: Don't skip versions

- If you're on v14, go v14 → v15 → v16, not v14 → v17
- Each version has its own migration schematics
```

---

## 4. Dependency Updates

### Update Strategy

```markdown
1. **Patch updates** (1.0.x) — Update freely, run tests. These should be safe.
2. **Minor updates** (1.x.0) — Read changelog, update, run tests. Watch for new deprecations.
3. **Major updates** (x.0.0) — Full migration assessment. Plan steps. Read migration guide.
```

### Handling Breaking Changes

#### Strategy 1: Adapter Pattern

Wrap the new API in the old interface. Migrate call sites one by one.

```typescript
// Old library API
import { oldClient } from 'old-http-lib'
oldClient.get(url, callback)

// New library has different API
import { newClient } from 'new-http-lib'
newClient.request({ url, method: 'GET' })

// Adapter — lets existing code keep working
class LegacyHttpAdapter {
  constructor(private client: typeof newClient) {}

  get(url: string, callback: (err: Error | null, data: any) => void): void {
    this.client
      .request({ url, method: 'GET' })
      .then(res => callback(null, res.data))
      .catch(err => callback(err, null))
  }

  post(url: string, body: any, callback: (err: Error | null, data: any) => void): void {
    this.client
      .request({ url, method: 'POST', data: body })
      .then(res => callback(null, res.data))
      .catch(err => callback(err, null))
  }
}

// Use adapter everywhere, then migrate call sites to new API one by one
// When all call sites are migrated, remove the adapter
```

#### Strategy 2: Codemods

Automated code transformation for mechanical changes across many files.

```typescript
// jscodeshift transform — rename method calls
// Run: npx jscodeshift -t transform.ts src/**/*.ts
import { API, FileInfo } from 'jscodeshift'

export default function transformer(file: FileInfo, api: API) {
  const j = api.jscodeshift
  return (
    j(file.source)
      // Find: oldMethod() → replace with: newMethod()
      .find(j.CallExpression, { callee: { name: 'oldMethod' } })
      .replaceWith(path => {
        return j.callExpression(j.identifier('newMethod'), path.node.arguments)
      })
      .toSource()
  )
}
```

#### Strategy 3: Feature Flag

Run old and new implementations side by side.

```typescript
class OrderService {
  constructor(
    private legacyPayment: LegacyPaymentService,
    private newPayment: NewPaymentService,
    private featureFlags: FeatureFlagService
  ) {}

  async processPayment(order: Order): Promise<PaymentResult> {
    if (this.featureFlags.isEnabled('new-payment-gateway', { userId: order.customerId })) {
      return this.newPayment.charge(order)
    }
    return this.legacyPayment.charge(order)
  }
}

// Roll out gradually: 1% → 10% → 50% → 100%
// Monitor error rates at each step
// If errors spike, disable the flag — instant rollback
```

### Peer Dependency Conflicts

```bash
# 1. Identify the conflict tree
npm ls [package-name]

# 2. Options in order of preference:
#    a. Update the parent package that has the peer dep requirement
#    b. Check if the peer dep range can be widened (read the lib's changelog)
#    c. Use npm overrides as TEMPORARY fix (document why and when to remove)
#    d. Fork and patch the library (last resort)

# npm overrides (package.json) — TEMPORARY FIX ONLY
{
  "overrides": {
    "some-package": {
      "problematic-dep": "^3.0.0"
    }
  }
}
# Always add a comment explaining why and a TODO to remove it
```

---

## 5. Data Migrations

### Batch Processing

```typescript
// Never migrate all data in one transaction — it locks the table and can OOM

// Generic batch processor with progress tracking
async function batchMigrate<T extends { id: string }>(options: {
  name: string
  query: () => Promise<T[]>
  process: (batch: T[]) => Promise<void>
  batchSize?: number
  onProgress?: (processed: number) => void
}): Promise<{ processed: number; duration: number }> {
  const { name, query, process, batchSize = 500, onProgress } = options
  const startTime = Date.now()
  let processed = 0

  console.log(`[${name}] Starting migration...`)

  while (true) {
    const batch = await query()
    if (batch.length === 0) break

    await process(batch)
    processed += batch.length
    onProgress?.(processed)

    console.log(`[${name}] Processed ${processed} records`)
  }

  const duration = Date.now() - startTime
  console.log(`[${name}] Complete. ${processed} records in ${duration}ms`)
  return { processed, duration }
}

// Usage with Prisma
await batchMigrate({
  name: 'migrate-order-status',
  query: () =>
    prisma.order.findMany({
      where: { legacyStatus: { not: null }, status: null },
      take: 500,
    }),
  process: async batch => {
    await prisma.$transaction(
      batch.map(order =>
        prisma.order.update({
          where: { id: order.id },
          data: {
            status: mapLegacyStatus(order.legacyStatus!),
            legacyStatus: null, // Clear after migration
          },
        })
      )
    )
  },
})

function mapLegacyStatus(legacy: number): OrderStatus {
  const map: Record<number, OrderStatus> = {
    0: 'DRAFT',
    1: 'PLACED',
    2: 'PROCESSING',
    3: 'SHIPPED',
    4: 'DELIVERED',
    9: 'CANCELLED',
  }
  const status = map[legacy]
  if (!status) throw new Error(`Unknown legacy status: ${legacy}`)
  return status
}
```

### Dual-Write Pattern

When migrating between data stores or changing data shape — write to both during transition.

```typescript
class DualWriteOrderRepository implements OrderRepository {
  constructor(
    private legacy: LegacyOrderRepository,
    private modern: ModernOrderRepository,
    private config: {
      readFromModern: boolean
      verifyConsistency: boolean
    }
  ) {}

  async save(order: Order): Promise<void> {
    // Always write to both — order doesn't matter but fail on either
    await Promise.all([this.legacy.save(order), this.modern.save(order)])
  }

  async findById(id: OrderId): Promise<Order | null> {
    if (this.config.readFromModern) {
      const modern = await this.modern.findById(id)

      // Optional: verify both stores agree during transition
      if (this.config.verifyConsistency) {
        const legacy = await this.legacy.findById(id)
        if (modern && legacy && !this.isConsistent(modern, legacy)) {
          console.error('Data inconsistency detected', { id: id.value })
        }
      }

      return modern
    }
    return this.legacy.findById(id)
  }

  private isConsistent(a: Order, b: Order): boolean {
    return a.status === b.status && a.total.equals(b.total) && a.items.length === b.items.length
  }
}

// Migration phases:
// Phase 1: readFromModern=false, verifyConsistency=false — just write to both
// Phase 2: readFromModern=false, verifyConsistency=true — verify data matches
// Phase 3: readFromModern=true, verifyConsistency=true — read from new, verify
// Phase 4: readFromModern=true, verifyConsistency=false — trust new store
// Phase 5: Remove legacy entirely
```

### Data Verification

```typescript
// After migration, verify data integrity
async function verifyMigration(options: {
  tableName: string
  expectedCount: number
  sampleSize: number
  verify: (record: any) => boolean
}): Promise<VerificationResult> {
  const { tableName, expectedCount, sampleSize, verify } = options

  // 1. Count check
  const count = await prisma.$queryRaw`SELECT COUNT(*) FROM ${tableName}`
  if (count !== expectedCount) {
    return { passed: false, reason: `Count mismatch: ${count} vs ${expectedCount}` }
  }

  // 2. Sample verification
  const sample = await prisma.$queryRaw`
    SELECT * FROM ${tableName} ORDER BY RANDOM() LIMIT ${sampleSize}
  `
  const failures = sample.filter((record: any) => !verify(record))
  if (failures.length > 0) {
    return { passed: false, reason: `${failures.length} records failed verification`, failures }
  }

  // 3. Null check on required fields
  const nullCheck = await prisma.$queryRaw`
    SELECT COUNT(*) FROM ${tableName} WHERE required_field IS NULL
  `
  if (nullCheck > 0) {
    return { passed: false, reason: `${nullCheck} records have NULL required fields` }
  }

  return { passed: true }
}
```

---

## 6. ORM/Database Migration

### Migrating Between ORMs

```markdown
## Strategy: TypeORM → Prisma (or any ORM switch)

### Phase 1: Add Prisma alongside TypeORM

- Install Prisma, generate schema from existing DB: `npx prisma db pull`
- Both ORMs point to same database
- New code uses Prisma, old code stays with TypeORM

### Phase 2: Migrate repositories one by one

- Replace one repository implementation at a time
- Keep the interface (OrderRepository) unchanged
- Test each migration independently

### Phase 3: Remove TypeORM

- Only after ALL repositories are migrated
- Remove TypeORM dependencies
- Clean up TypeORM config files
```

### Migrating Between Databases

```markdown
## Strategy: MongoDB → PostgreSQL (or any DB switch)

### Phase 1: Shadow writes

- Set up PostgreSQL with schema
- Write to both MongoDB and PostgreSQL
- Read from MongoDB only

### Phase 2: Backfill historical data

- Batch-migrate all existing data from MongoDB to PostgreSQL
- Verify counts and data integrity

### Phase 3: Shadow reads

- Read from PostgreSQL
- Compare results with MongoDB (log mismatches)
- Fix any data or query inconsistencies

### Phase 4: Switch reads

- Read from PostgreSQL only
- Keep writing to MongoDB as backup

### Phase 5: Remove MongoDB

- Stop writing to MongoDB
- Remove MongoDB dependencies
- Keep backup for safety window (e.g., 30 days)
```

---

## 7. Anti-Patterns

- **Big bang migration** — Changing everything at once. One failure rolls back everything. Migrate incrementally.
- **No rollback plan** — Every step needs a way back. "We'll fix it forward" is not a plan.
- **Mixing migration with feature work** — Migration PRs should contain ONLY migration changes. No "while I'm here" improvements.
- **Ignoring the changelog** — Read the FULL migration guide. Behavioral changes hide in patch notes and are the most dangerous.
- **Testing only happy paths** — Test error cases, edge cases, and partial failure scenarios. What happens if the migration fails halfway?
- **Skipping data verification** — After data migration, verify counts, checksums, and sample records.
- **No monitoring during rollout** — Watch error rates, latency, and data integrity during and after migration.
- **Skipping versions** — Going from v2 to v5 directly. Each major version has its own migration path.
- **Migrating on Friday** — Never deploy migrations before weekends or holidays.
- **Deleting before confirming** — Don't drop old columns/tables until the new path is proven in production.

---

## 8. Output

```
## Migration Plan: [from] → [to]

### Assessment
- Breaking changes: X
- Files affected: X
- Risk level: [Low/Medium/High/Critical]
- Estimated deploys: X

### Steps
1. [Step] — Rollback: [how]
2. [Step] — Rollback: [how]
3. [Step] — Rollback: [how]
   ⚠ Point of no return after this step

### Verification
- [ ] All tests pass
- [ ] Build succeeds
- [ ] No runtime errors in [scope]
- [ ] Data integrity verified (count, sample, nulls)
- [ ] Performance unchanged (latency, query times)

### Monitoring
- [ ] Error rates baseline recorded
- [ ] Alerts configured for anomalies
- [ ] Rollback procedure documented and tested

### Cleanup (post-migration, after stability window)
- [ ] Remove compatibility code
- [ ] Remove old dependencies
- [ ] Drop old columns/tables
- [ ] Remove feature flags
- [ ] Update documentation
```

## Inspired By

- **Release It!** — Michael Nygard (Stability Patterns)
- **Refactoring Databases** — Scott Ambler & Pramod Sadalage
- **Building Evolutionary Architectures** — Neal Ford, Rebecca Parsons, Patrick Kua
- **Designing Data-Intensive Applications** — Martin Kleppmann
- **Database Reliability Engineering** — Laine Campbell & Charity Majors
