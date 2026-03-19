---
name: sfermanelli-debug
description: Investigate and fix bugs by tracing root causes, not just symptoms. Use this skill when the user reports a bug, error, unexpected behavior, or something that "doesn't work". Triggers for "this is broken", "why isn't this working", "I'm getting an error", "it crashes when", "unexpected behavior", "this used to work", "can you debug this", stack traces, error screenshots, or any problem-solving scenario. Different from fix-lint (which is automated) — this is investigation and diagnosis.
---

# Debug

Find and fix bugs by understanding the root cause — not by guessing, not by adding try-catch everywhere, and definitely not by rewriting the whole thing.

> "Debugging is twice as hard as writing the code in the first place. Therefore, if you write the code as cleverly as possible, you are, by definition, not smart enough to debug it." — Brian Kernighan

## Golden Rule

**Don't guess — observe.** Every theory must be backed by evidence. Add a log, check the data, read the stack trace. The bug is in the code, not in the framework.

---

## 1. Investigation Process

### Step 1: Reproduce

Before fixing anything, understand the problem:

- What's the expected behavior?
- What's the actual behavior?
- What are the exact steps to reproduce?
- Does it happen consistently or intermittently?
- Does it happen in all environments (dev, staging, prod)?
- What changed recently? (`git log --oneline -20`)

```bash
# Find what changed recently
git log --oneline -20
git diff HEAD~5 --stat

# Find when a bug was introduced
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
# Git will binary-search through commits
# Test each one, mark as good/bad
git bisect good  # or git bisect bad
# When found:
git bisect reset
```

### Step 2: Isolate

Narrow down where the bug lives:

```
Frontend?  →  Check browser console, network tab, React DevTools
Backend?   →  Check server logs, request/response payloads
Database?  →  Check query results, RLS policies, constraints
Infra?     →  Check env vars, DNS, certificates, deployment logs
```

**Boundaries are bug magnets** — most bugs hide at the seams between systems:

| Boundary               | Common bugs                                                            |
| ---------------------- | ---------------------------------------------------------------------- |
| Client ↔ API           | Wrong request format, missing headers, CORS, auth token expired        |
| API ↔ Database         | Column renamed, RLS blocking, type mismatch, missing migration         |
| Server ↔ Client render | Hydration mismatch, SSR using browser APIs, stale cache                |
| Service ↔ Service      | Timeout, schema change, version mismatch, network partition            |
| Code ↔ Config          | Missing env var, wrong env var value, different config per environment |

### Step 3: Diagnose

Identify the root cause, not just the symptom:

```
Symptom:     "The page shows a 500 error"
Surface:     "The API route throws an error"
Root cause:  "The query uses `renter_id` but the column was renamed to `client_id`
              in migration 00038"
```

**Ask "why" five times** (Toyota's 5 Whys):

```
1. Why did the page show 500?        → The API threw an unhandled error
2. Why was the error unhandled?       → The query failed
3. Why did the query fail?            → Column `renter_id` doesn't exist
4. Why doesn't the column exist?      → Migration 00038 renamed it to `client_id`
5. Why wasn't the code updated?       → The rename was in a different PR, code wasn't searched
```

The fix for cause #5 (add a grep check to the migration process) is better than the fix for cause #1 (add a try-catch).

### Step 4: Fix

Apply the **minimal change** that fixes the root cause:

- Don't refactor while debugging — fix the bug, then clean up separately
- Don't add defensive code that hides the real problem
- If the fix is more than ~20 lines, reconsider — you might be fixing the wrong thing
- Write a test that reproduces the bug BEFORE fixing it

### Step 5: Verify

- Confirm the fix resolves the original issue
- Check for regressions — did fixing this break something else?
- Consider edge cases — does the fix work for all inputs, not just the reported one?
- Run the test suite

---

## 2. Debugging Tools

### Browser DevTools

```
Console tab:
  - Errors, warnings, logs
  - filter by log level or source
  - console.table() for data inspection
  - console.trace() for call stack

Network tab:
  - Request/response payloads
  - Status codes and timing
  - Filter by XHR/Fetch to see API calls
  - Check "Preserve log" for navigation-based bugs

Sources tab:
  - Set breakpoints in source code
  - Conditional breakpoints (right-click → "Add conditional breakpoint")
  - Watch expressions
  - Call stack inspection
  - Step over (F10), Step into (F11), Step out (Shift+F11)

Application tab:
  - LocalStorage, SessionStorage, Cookies
  - Service Workers, Cache Storage
  - IndexedDB

Performance tab:
  - Record → reproduce the issue → Stop
  - Look for long tasks (>50ms), layout shifts, excessive re-renders

React DevTools:
  - Components tab: inspect props, state, hooks
  - Profiler tab: record renders, see what re-rendered and why
  - Highlight updates: Settings → "Highlight updates when components render"

Angular DevTools:
  - Component Explorer: inspect component tree, inputs, outputs
  - Profiler: detect change detection cycles
  - Enable in Chrome: install Angular DevTools extension
```

### Node.js Debugging

```bash
# Start Node.js with inspector
node --inspect src/server.ts
# Opens debugger on ws://127.0.0.1:9229
# Connect via chrome://inspect or VS Code

# NestJS debugging
# In package.json:
"debug": "node --inspect-brk -r ts-node/register src/main.ts"
# Or in VS Code launch.json:
{
  "type": "node",
  "request": "launch",
  "name": "Debug NestJS",
  "runtimeArgs": ["--nolazy", "-r", "ts-node/register"],
  "args": ["src/main.ts"],
  "sourceMaps": true
}
```

### VS Code Debugger

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      // Debug current test file
      "type": "node",
      "request": "launch",
      "name": "Debug Current Test",
      "program": "${workspaceFolder}/node_modules/.bin/vitest",
      "args": ["run", "${relativeFile}"],
      "console": "integratedTerminal",
    },
    {
      // Debug Next.js server
      "type": "node",
      "request": "launch",
      "name": "Debug Next.js",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "port": 9229,
      "console": "integratedTerminal",
    },
  ],
}
```

### Database Debugging

```sql
-- Check what a query actually does (PostgreSQL)
EXPLAIN ANALYZE
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 20;

-- Key things to look for:
-- Seq Scan on large tables → missing index
-- Nested Loop with high row count → N+1 or bad join
-- Sort with high cost → missing index on ORDER BY column
-- Actual rows vs planned rows mismatch → stale statistics (ANALYZE the table)

-- Check RLS policies (Supabase)
-- As admin (bypasses RLS):
SELECT * FROM orders WHERE id = 'ord_123';
-- As user (applies RLS):
SET request.jwt.claims = '{"sub": "user_123"}';
SELECT * FROM orders WHERE id = 'ord_123';
-- If results differ → RLS policy is blocking

-- Check active connections and locks
SELECT pid, state, query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

### Strategic Logging

```typescript
// Temporary debug logging — REMOVE AFTER DEBUGGING
// Use a prefix to find and remove them easily

// Bad — generic, hard to find later
console.log(data)
console.log('here')

// Good — prefixed, contextual, easy to grep and remove
console.log('[DEBUG:order-create]', { customerId, items: items.length, total })
console.log('[DEBUG:auth]', { userId: user?.id, role: user?.role, hasToken: !!token })
console.log('[DEBUG:query]', { sql: query.toSQL(), params })

// For timing
console.time('[DEBUG:order-processing]')
await processOrder(order)
console.timeEnd('[DEBUG:order-processing]')
// Output: [DEBUG:order-processing]: 342ms

// After debugging, find and remove all:
// grep -rn '\[DEBUG:' src/
```

---

## 3. Common Bug Patterns

### Async / Timing

```typescript
// Bug: Missing await — gets a Promise instead of the value
const user = getUser(id) // Promise<User>, not User
console.log(user.name) // undefined
// Fix:
const user = await getUser(id)

// Bug: Race condition — parallel writes to same resource
await updateStock(productId, -1) // Two requests hit simultaneously
await updateStock(productId, -1) // Both read stock=10, both write stock=9 (should be 8)
// Fix: Use database transaction or optimistic locking
await prisma.$transaction(async tx => {
  const product = await tx.product.findUnique({ where: { id: productId } })
  await tx.product.update({
    where: { id: productId, stock: product.stock }, // Optimistic lock
    data: { stock: product.stock - 1 },
  })
})

// Bug: Stale closure — captures old state value
function Counter() {
  const [count, setCount] = useState(0)
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1) // Always reads the initial count (0)
    }, 1000)
    return () => clearInterval(interval)
  }, []) // Empty deps → closure captures count=0 forever
  // Fix:
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(prev => prev + 1) // Functional update, no closure dependency
    }, 1000)
    return () => clearInterval(interval)
  }, [])
}

// Bug: Event listener not cleaned up → memory leak
useEffect(() => {
  window.addEventListener('resize', handleResize)
  // Missing cleanup! Listener stays after unmount
}, [])
// Fix:
useEffect(() => {
  window.addEventListener('resize', handleResize)
  return () => window.removeEventListener('resize', handleResize)
}, [])

// Bug: Promise.all fails fast — one rejection kills all
const [users, orders] = await Promise.all([getUsers(), getOrders()])
// If getOrders() fails, users result is lost too
// Fix — when you want partial results:
const [usersResult, ordersResult] = await Promise.allSettled([getUsers(), getOrders()])
const users = usersResult.status === 'fulfilled' ? usersResult.value : []
```

### Type Coercion / JavaScript Gotchas

```typescript
// Bug: == instead of ===
0 == ''           // true
0 == '0'          // true
'' == '0'         // false (inconsistent!)
null == undefined // true
// Fix: always use ===

// Bug: Truthy/falsy confusion
const count = 0
if (count) { ... }  // Doesn't run! 0 is falsy
// Fix: be explicit
if (count !== undefined && count !== null) { ... }
// Or:
if (count != null) { ... }  // Only case where == is acceptable (null/undefined check)

// Bug: parseInt without radix
parseInt('08')     // 8 in modern engines, 0 in old ones
parseInt('0x10')   // 16 (parsed as hex!)
// Fix:
parseInt('08', 10)  // Always specify radix

// Bug: Floating point
0.1 + 0.2 === 0.3  // false (0.30000000000000004)
// Fix: use integer cents for money, or compare with epsilon
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON

// Bug: Array sort without comparator
[10, 9, 8, 1, 2, 3].sort()  // [1, 10, 2, 3, 8, 9] — lexicographic!
// Fix:
[10, 9, 8, 1, 2, 3].sort((a, b) => a - b)  // [1, 2, 3, 8, 9, 10]

// Bug: typeof null
typeof null === 'object'  // true (historical JS bug)
// Fix: explicit null check
if (value !== null && typeof value === 'object') { ... }
```

### Database

```typescript
// Bug: Column renamed but code not updated
// Error: "column orders.renter_id does not exist"
// Diagnosis: grep for the old column name
// grep -rn 'renter_id' src/

// Bug: RLS policy blocking access
// Symptom: Works with service key, returns empty with user key
// Diagnosis: Check the policy's USING clause
// SELECT * FROM pg_policies WHERE tablename = 'orders';

// Bug: Missing index causing timeout
// Symptom: Query takes 30s+ on large table
// Diagnosis: EXPLAIN ANALYZE the query, look for Seq Scan

// Bug: FK constraint preventing delete
// Error: "update or delete violates foreign key constraint"
// Diagnosis: Check what references this row
// SELECT tc.table_name, kcu.column_name
// FROM information_schema.table_constraints tc
// JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
// WHERE tc.constraint_type = 'FOREIGN KEY' AND ccu.table_name = 'orders';

// Bug: Transaction deadlock
// Symptom: Query hangs, eventually times out with "deadlock detected"
// Diagnosis: Two transactions lock the same rows in different order
// Fix: Always lock rows in the same order, or use shorter transactions

// Bug: Prisma — relation not loaded
const order = await prisma.order.findUnique({ where: { id } })
console.log(order.items) // undefined! Relations aren't loaded by default
// Fix:
const order = await prisma.order.findUnique({
  where: { id },
  include: { items: true }, // Explicitly include relations
})
```

### React

```typescript
// Bug: Missing key prop → stale renders, wrong items
{items.map(item => <Item {...item} />)}  // No key!
// Fix:
{items.map(item => <Item key={item.id} {...item} />)}
// Never use array index as key if list can reorder/filter

// Bug: useEffect missing dependency → stale data
useEffect(() => {
  fetchOrders(storeId)  // storeId changes but effect doesn't re-run
}, [])
// Fix:
useEffect(() => {
  fetchOrders(storeId)
}, [storeId])

// Bug: Server component using client API
// Error: "useState is not a function" or "window is not defined"
// Server components can't use hooks, browser APIs, or event handlers
// Fix: Add "use client" directive or split into server + client components

// Bug: Hydration mismatch
// Warning: "Text content did not match"
// Caused by: rendering different content on server vs client
// Common causes: Date.now(), Math.random(), window.innerWidth, user-agent sniffing
// Fix: use useEffect for client-only values, or suppressHydrationWarning

// Bug: Infinite re-render loop
useEffect(() => {
  setData(transform(rawData))  // setData triggers re-render → effect runs again
}, [rawData, data])  // data in deps creates the loop
// Fix: remove data from deps, or use useMemo instead of useEffect

// Bug: Object/array in dependency array
useEffect(() => { ... }, [{ id: 1 }])  // New object every render → infinite loop
// Fix: depend on primitives, or useMemo the object
useEffect(() => { ... }, [id])
```

### Angular

```typescript
// Bug: Change detection not triggering
// Symptom: Data updated but view doesn't reflect it
// Cause: Mutating object/array reference with OnPush strategy
this.items.push(newItem)  // Same reference → OnPush doesn't detect
// Fix: Create new reference
this.items = [...this.items, newItem]

// Bug: Memory leak from unsubscribed Observable
ngOnInit() {
  this.dataService.getData().subscribe(data => this.data = data)
  // Never unsubscribed → stays alive after component destroy
}
// Fix:
private destroy$ = new Subject<void>()
ngOnInit() {
  this.dataService.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => this.data = data)
}
ngOnDestroy() {
  this.destroy$.next()
  this.destroy$.complete()
}

// Bug: ExpressionChangedAfterItHasBeenCheckedError
// Cause: Changing a value during change detection
ngAfterViewInit() {
  this.title = 'New Title'  // Changes after CD ran → error in dev mode
}
// Fix: wrap in setTimeout or use ChangeDetectorRef
ngAfterViewInit() {
  this.cdr.detectChanges()  // or setTimeout(() => this.title = 'New Title')
}

// Bug: Circular dependency injection
// Error: "Cannot instantiate cyclic dependency"
// Cause: ServiceA depends on ServiceB, ServiceB depends on ServiceA
// Fix: Break the cycle with an event bus, or use forwardRef(), or refactor
```

### NestJS

```typescript
// Bug: Injection token not found
// Error: "Nest can't resolve dependencies of ServiceX (?)"
// Cause: Missing provider in module, or circular dependency
// Fix: Check @Module({ providers: [...] }) includes the dependency
// For circular: use forwardRef(() => ServiceB)

// Bug: Guard/interceptor order matters
// Symptom: Auth guard runs after validation pipe
// NestJS execution order: Middleware → Guards → Interceptors (pre) → Pipes → Handler → Interceptors (post) → Exception filters
// Fix: Know the order, place logic in the right layer

// Bug: Async provider not awaited
// Symptom: Service is undefined or partially initialized
// Fix: Use async factory providers
{
  provide: 'DATABASE',
  useFactory: async () => {
    const connection = await createConnection(config)
    return connection
  }
}

// Bug: Validation pipe not applied globally
// Symptom: DTOs aren't validated, invalid data reaches handlers
// Fix: Apply globally in main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,        // Strip unknown properties
  forbidNonWhitelisted: true,  // Throw if unknown properties
  transform: true         // Auto-transform types
}))
```

### Next.js

```typescript
// Bug: headers()/cookies() not awaited (Next.js 15+)
// Error: "Route used headers without awaiting"
const headersList = headers() // Returns Promise in Next.js 15+
// Fix:
const headersList = await headers()

// Bug: Dynamic import needed for client-only libraries
// Error: "window is not defined" or "document is not defined"
import Leaflet from 'leaflet' // Fails during SSR
// Fix:
const Leaflet = dynamic(() => import('leaflet'), { ssr: false })

// Bug: Middleware running on routes it shouldn't
// Symptom: Static assets blocked, API routes intercepted
// Fix: Configure matcher in middleware.ts
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|api/webhooks).*)'],
}

// Bug: Stale cache after data mutation
// Symptom: Page shows old data after create/update
// Fix: Revalidate after mutation
import { revalidatePath, revalidateTag } from 'next/cache'
async function createOrder(data: CreateOrderInput) {
  await db.order.create({ data })
  revalidatePath('/orders') // Revalidate specific path
  revalidateTag('orders') // Or revalidate by tag
}

// Bug: Route handler caching GET responses
// Symptom: GET endpoint returns stale data
// Next.js caches GET route handlers by default
// Fix: opt out of caching
export const dynamic = 'force-dynamic'
// Or use headers/cookies to make it dynamic automatically
```

---

## 4. Error Message Decoder

Common cryptic error messages and what they actually mean:

| Error Message                                            | Actual Cause                                                             | Fix                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `Cannot read properties of undefined (reading 'x')`      | Variable is undefined, you're accessing `.x` on it                       | Add null check, trace where value should come from                    |
| `TypeError: X is not a function`                         | You're calling something that isn't a function (wrong import, undefined) | Check import, check that the value is actually a function             |
| `Hydration failed because the initial UI does not match` | Server rendered different HTML than client                               | Move dynamic content to useEffect, check for Date/random/window usage |
| `Maximum update depth exceeded`                          | Infinite re-render loop (setState in render, circular useEffect)         | Check useEffect deps, don't setState during render                    |
| `Cannot find module 'X'`                                 | Package not installed, wrong import path, missing file extension         | `npm install X`, check path, check tsconfig paths                     |
| `ECONNREFUSED`                                           | Service isn't running on that port                                       | Check if server is running, check port, check firewall                |
| `ETIMEOUT / ETIMEDOUT`                                   | Connection took too long                                                 | Server overloaded, network issue, missing timeout config              |
| `P2002 (Prisma)`                                         | Unique constraint violation                                              | Check for duplicates, handle conflict gracefully                      |
| `P2025 (Prisma)`                                         | Record not found for update/delete                                       | Check if record exists before operating                               |
| `NEXT_NOT_FOUND`                                         | Next.js notFound() was called                                            | Check data fetching, route params, revalidation                       |
| `42501 (PostgreSQL)`                                     | Permission denied (RLS)                                                  | Check RLS policies, check JWT claims                                  |
| `23505 (PostgreSQL)`                                     | Unique constraint violation                                              | Duplicate key, handle upsert or conflict                              |
| `NG0100 (Angular)`                                       | ExpressionChangedAfterItHasBeenChecked                                   | Don't modify state in AfterViewInit, use detectChanges()              |
| `NG0200 (Angular)`                                       | Circular dependency in DI                                                | Break cycle with forwardRef or refactor                               |
| `NullInjectorError (Angular)`                            | Provider not found in DI tree                                            | Add to module providers, check import of module                       |

---

## 5. Debugging Strategies

### Binary Search (The Wolf Fence)

When you don't know where the bug is, divide and conquer:

```typescript
// Strategy 1: Code bisection — add a log in the middle
async function processOrder(order: Order) {
  const validated = validate(order)
  console.log('[DEBUG] After validate:', validated)  // Is the data correct here?

  const priced = calculatePricing(validated)
  console.log('[DEBUG] After pricing:', priced)       // And here?

  const saved = await saveOrder(priced)
  console.log('[DEBUG] After save:', saved)            // And here?

  await sendConfirmation(saved)
}
// If data is wrong after pricing but correct after validate → bug is in calculatePricing

// Strategy 2: Git bisect — find the commit that broke it
git bisect start
git bisect bad          # Current version is broken
git bisect good v1.2.0  # This version worked
# Git checks out a middle commit
# Test it, then:
git bisect good  # or git bisect bad
# Repeat until git identifies the exact commit
git bisect reset

// Strategy 3: Comment-out bisection — for UI bugs
// Comment out half the JSX, does the bug still appear?
// Yes → bug is in the remaining half
// No → bug is in the commented-out half
// Repeat until isolated
```

### Rubber Duck Debugging

Explain the problem step by step, out loud or in writing. The act of articulating forces you to confront assumptions:

```markdown
"The order total should be $49.99.
The items array has 2 items: $19.99 and $29.99. That's $49.98, not $49.99.
Wait — there's a $0.01 rounding issue.
Actually, the total includes a $0.01 service fee that I forgot about.
...never mind, the code is correct. The bug is in my test expectation."
```

### Diff Debugging

When something "used to work":

```bash
# What changed in the last day?
git log --since="1 day ago" --oneline

# What changed in a specific file?
git log --oneline -10 -- src/services/order.service.ts

# Show the exact diff of a suspicious commit
git show abc123

# Compare current code with a known-good version
git diff v1.2.0..HEAD -- src/services/order.service.ts
```

---

## 6. Debug Output

```markdown
## Bug Report

**Symptom:** [what the user sees]
**Root cause:** [the actual problem]
**Fix:** [what was changed and why]

### Investigation trace:

1. [first thing checked and what was found]
2. [second thing checked]
3. [the "aha" moment — where the root cause was identified]

### Files changed:

- `path/to/file.ts:42` — [what was changed]

### Test added:

- `path/to/file.test.ts` — reproduces the bug, verifies the fix

### Verified:

- ✓ Original issue resolved
- ✓ No regressions detected
- ✓ Edge cases considered
```

---

## 7. Principles

### Core Rules

- **Read before writing** — understand the code before changing it
- **One fix at a time** — don't bundle unrelated changes with the bug fix
- **Don't guess** — if you're not sure, add a log and reproduce. Guessing wastes time.
- **The simplest explanation is usually right** — before suspecting a framework bug, check your code

### From The Pragmatic Programmer

- **"select() isn't broken"** — it's almost certainly YOUR code, not the framework, OS, or compiler
- **"Don't assume it, prove it"** — verify every assumption with evidence
- **Rubber duck debugging** — explain the problem step by step. The act of explaining often reveals the answer.
- **"Find bugs once"** — once a human finds a bug, write a test. It should be the last time a human finds that bug.

### Agans' 9 Rules of Debugging

From **Debugging** by David Agans:

1. **Understand the system** — read the docs, know the architecture
2. **Make it fail** — find a reliable reproduction
3. **Quit thinking and look** — observe actual behavior, don't theorize
4. **Divide and conquer** — narrow down systematically (binary search)
5. **Change one thing at a time** — scientific method
6. **Keep an audit trail** — write down what you tried and what happened
7. **Check the plug** — verify the obvious first (is the server running? is the env var set?)
8. **Get a fresh view** — ask someone else, or step away and come back
9. **If you didn't fix it, it ain't fixed** — verify the fix, don't assume

### Stability Patterns (Release It! — Michael Nygard)

When debugging production issues, understand failure cascades:

- **Bulkheads** — isolate failures so one broken service doesn't take everything down
- **Circuit breakers** — stop calling a failing service, fail fast instead
- **Timeouts** — always set timeouts on external calls. No timeout = infinite hang risk
- **Fail fast** — if you know something is wrong, throw immediately. Don't let bad data propagate.

## Inspired By

- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **Debugging** — David Agans (9 Rules of Debugging)
- **Release It!** — Michael Nygard (Stability Patterns)
- **Why Programs Fail** — Andreas Zeller
- **Effective Debugging** — Diomidis Spinellis
