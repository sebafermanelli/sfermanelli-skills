---
name: sfermanelli-fix-lint
description: Automatically fix linting and type errors across the codebase. Use this skill when the user asks to fix lint errors, fix ESLint, fix TypeScript errors, fix type errors, resolve warnings, clean up lint, or says things like "fix the red squiggles", "make it pass lint", "fix tsc errors", "resolve these warnings", "npm run lint has errors". Also triggers for "fix build errors" when they're type-related.
---

# Fix Lint

Fix linting and type errors efficiently — understand the root cause, apply the correct fix, not a band-aid.

## Golden Rule

**Fix the actual code, not the symptom.** An `as any` is not a fix — it's duct tape. If the types are wrong, fix the types. If the code is wrong, fix the code. Suppressions are the last resort.

---

## 1. Process

1. **Run the linter/type checker** — detect the project's tools:
   ```bash
   npm run lint          # or npx eslint .
   npx tsc --noEmit      # TypeScript type check without emitting
   ng lint               # Angular projects
   ```
2. **Parse the errors** — group by type, not by file. One root cause often produces many errors.
3. **Fix root causes first** — fixing a wrong type definition might resolve 20 errors at once.
4. **Re-run to verify** — confirm zero errors after fixes.
5. **Report** — what was fixed and why.

## 2. Fix Hierarchy

Always prefer higher over lower:

```
1. Fix the actual code      — the code is wrong, fix the logic
2. Add proper types          — missing type annotations, incorrect generics
3. Handle the edge case      — null check, undefined guard, optional chaining
4. Narrow the type           — type guard, discriminated union, assertion function
5. Update the config         — rule is too strict or doesn't apply to this project
6. Suppress with comment     — LAST RESORT, only with explanation
```

---

## 3. TypeScript Error Patterns

### Assignment Errors

```typescript
// TS2322: Type 'X' is not assignable to type 'Y'
// Cause: Variable or return type doesn't match expected type

// Example: string vs string literal
let status: 'active' | 'inactive' = 'active'
const fromApi: string = getStatus()
status = fromApi // Error: string not assignable to 'active' | 'inactive'
// Fix: validate and narrow
if (fromApi === 'active' || fromApi === 'inactive') {
  status = fromApi // OK — narrowed
}
// Or: use Zod to parse at the boundary
const StatusSchema = z.enum(['active', 'inactive'])
status = StatusSchema.parse(fromApi)
```

### Property Errors

```typescript
// TS2339: Property 'x' does not exist on type 'Y'
// Cause: Accessing a property that isn't in the type definition

// Fix 1: Add to the interface
interface User {
  name: string
  email: string
  avatarUrl?: string // Add the missing property
}

// Fix 2: Use optional chaining (if the property might not exist)
const url = user?.avatarUrl ?? '/default-avatar.png'

// Fix 3: Type guard for union types
function isAdmin(user: User | Admin): user is Admin {
  return 'permissions' in user
}
if (isAdmin(user)) {
  console.log(user.permissions) // OK — narrowed to Admin
}
```

### Null / Undefined Errors

```typescript
// TS2531: Object is possibly 'null'
// TS2532: Object is possibly 'undefined'
// Cause: strictNullChecks is on and you're not handling null

// Fix 1: Null check (preferred)
const user = await getUser(id)
if (!user) {
  throw new NotFoundError('User', id)
}
// After this point, user is narrowed to non-null

// Fix 2: Optional chaining (for optional access)
const name = user?.profile?.displayName ?? 'Anonymous'

// Fix 3: Non-null assertion (only when you're 100% certain)
const element = document.getElementById('root')!
// Use sparingly — this tells TypeScript "trust me, it's not null"
// If you're wrong, you get a runtime error

// Fix 4: Assertion function (validates and narrows)
function assertDefined<T>(value: T | null | undefined, name: string): asserts value is T {
  if (value == null) throw new Error(`${name} is required but was ${value}`)
}
const user = await getUser(id)
assertDefined(user, 'user')
// user is now narrowed to User (non-null)
```

### Generic Errors

```typescript
// TS2344: Type 'X' does not satisfy the constraint 'Y'
// Cause: Generic type parameter doesn't meet the constraint

// Example
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
getProperty(user, 'nonExistent')  // Error: '"nonExistent"' not assignable to keyof User

// TS2345: Argument of type 'X' is not assignable to parameter of type 'Y'
// Cause: Function argument doesn't match parameter type

// Common with callbacks and event handlers
document.addEventListener('click', (e: MouseEvent) => { ... })
// If you pass an incompatible handler type, widen or narrow appropriately
```

### Module Errors

```typescript
// TS2307: Cannot find module 'X'
// Cause: Package not installed, wrong path, missing type declarations

// Fix 1: Install the package
npm install package-name

// Fix 2: Install type declarations
npm install -D @types/package-name

// Fix 3: Create a declaration file for untyped modules
// src/types/untyped-lib.d.ts
declare module 'untyped-lib' {
  export function doSomething(input: string): void
  export default function main(): void
}

// Fix 4: Fix the import path
import { thing } from './utils'     // Relative path
import { thing } from '@/utils'     // Path alias (check tsconfig paths)
import { thing } from 'package'     // Node module
```

### Async Errors

```typescript
// TS2801: This expression is not callable (Type 'Promise<X>' has no call signatures)
// Cause: Missing await — you're trying to use a Promise as if it were the resolved value

const user = getUser(id) // Forgot await — user is Promise<User>
user.name // Error: Property 'name' does not exist on Promise
// Fix:
const user = await getUser(id)

// TS1062: Type is referenced directly or indirectly in the fulfillment callback
// Cause: Circular type reference in async code
// Fix: Break the cycle by extracting a type alias
```

### Excess Property Checks

```typescript
// TS2353: Object literal may only specify known properties
const config: Config = {
  port: 3000,
  hosst: 'localhost', // Error: typo! 'hosst' doesn't exist in Config
}

// This only applies to object literals — not to variables
const raw = { port: 3000, hosst: 'localhost' }
const config: Config = raw // No error! (excess properties allowed from variables)
// This is why Zod validation at boundaries is important
```

### Type Narrowing Patterns

```typescript
// Use narrowing instead of casting — it's checked at runtime

// typeof guard
function process(value: string | number) {
  if (typeof value === 'string') {
    return value.toUpperCase() // TypeScript knows it's string here
  }
  return value.toFixed(2) // TypeScript knows it's number here
}

// in guard
function handle(shape: Circle | Rectangle) {
  if ('radius' in shape) {
    return Math.PI * shape.radius ** 2 // Circle
  }
  return shape.width * shape.height // Rectangle
}

// instanceof guard
if (error instanceof ValidationError) {
  return res.status(400).json({ fields: error.fields })
}

// Discriminated union
type Result = { ok: true; data: Order } | { ok: false; error: AppError }
function handle(result: Result) {
  if (result.ok) {
    console.log(result.data) // TypeScript knows it's { ok: true; data: Order }
  } else {
    console.log(result.error) // TypeScript knows it's { ok: false; error: AppError }
  }
}

// Custom type guard
function isNonNull<T>(value: T | null | undefined): value is T {
  return value != null
}
const validUsers = users.filter(isNonNull) // Type: User[] (not (User | null)[])
```

---

## 4. ESLint Error Patterns

### React Hooks

```typescript
// react-hooks/exhaustive-deps — missing dependency in useEffect
useEffect(() => {
  fetchData(userId)
}, [])  // Warning: 'userId' is missing from deps

// Fix 1: Add the dependency (most common fix)
useEffect(() => {
  fetchData(userId)
}, [userId])

// Fix 2: Move the function inside useEffect (if it's only used there)
useEffect(() => {
  const fetchData = async () => { ... }
  fetchData()
}, [userId])

// Fix 3: Suppress with explanation (rare — only for intentional fire-once)
useEffect(() => {
  initializeAnalytics()  // Truly should only run once, regardless of deps
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [])
```

### Unused Variables

```typescript
// @typescript-eslint/no-unused-vars
// Fix 1: Remove the variable
// Fix 2: Prefix with _ if intentionally unused (destructuring rest, callback params)
const { unusedField, ...rest } = data // Error on unusedField
const { _unusedField, ...rest } = data // OK with _ prefix

array.map((_item, index) => index) // _ prefix for unused callback params
```

### Import Ordering

```typescript
// import/order — imports in wrong order
// Most projects want: external → internal → relative → styles

// External packages
import { z } from 'zod'
import { Injectable } from '@nestjs/common'

// Internal aliases
import { OrderService } from '@/services/order.service'
import { Money } from '@/domain/value-objects/money'

// Relative imports
import { OrderMapper } from './order.mapper'
import { CreateOrderDto } from './dto/create-order.dto'

// Styles/assets (if applicable)
import './styles.css'
```

### Common ESLint Fixes

| Rule                                        | Error                                      | Fix                                                   |
| ------------------------------------------- | ------------------------------------------ | ----------------------------------------------------- |
| `no-explicit-any`                           | Using `any` type                           | Add proper type. If truly unknown, use `unknown`.     |
| `prefer-const`                              | `let` used but never reassigned            | Change `let` to `const`                               |
| `no-unused-vars`                            | Variable declared but never used           | Remove it, or prefix with `_`                         |
| `eqeqeq`                                    | Using `==` instead of `===`                | Use `===` (except `!= null` for null/undefined check) |
| `no-console`                                | `console.log` in production code           | Remove, or use proper logger                          |
| `@typescript-eslint/no-floating-promises`   | Promise not awaited or handled             | Add `await` or `.catch()`                             |
| `@typescript-eslint/no-misused-promises`    | Passing async function where sync expected | Wrap in non-async function, or fix the type           |
| `react/no-unescaped-entities`               | `'` or `"` in JSX                          | Use `&apos;` or `{'\''}`                              |
| `@angular-eslint/no-empty-lifecycle-method` | Empty ngOnInit etc.                        | Remove the empty method                               |
| `@angular-eslint/use-lifecycle-interface`   | Missing implements OnInit                  | Add `implements OnInit` to class                      |

---

## 5. TSConfig Strict Options

### What Each Flag Does

```jsonc
{
  "compilerOptions": {
    // The "strict" umbrella enables all of these:
    "strict": true,

    // Individual flags (for incremental adoption):
    "strictNullChecks": true, // null and undefined are their own types
    "strictFunctionTypes": true, // Stricter function parameter checking
    "strictBindCallApply": true, // Check bind/call/apply arguments
    "strictPropertyInitialization": true, // Class properties must be initialized
    "noImplicitAny": true, // Can't have implicit 'any' type
    "noImplicitThis": true, // 'this' must have a type
    "alwaysStrict": true, // Emit "use strict" in output

    // Extra strictness (not included in "strict"):
    "noUncheckedIndexedAccess": true, // Array/object index access returns T | undefined
    "exactOptionalPropertyTypes": true, // Can't assign undefined to optional props
    "noImplicitReturns": true, // Every code path must return a value
    "noFallthroughCasesInSwitch": true, // Switch cases must break or return
    "noPropertyAccessFromIndexSignature": true, // Must use bracket notation for index signatures
  },
}
```

### Incremental Strict Migration

```bash
# Step 1: Enable strict in tsconfig, see how many errors
npx tsc --noEmit 2>&1 | wc -l

# Step 2: If too many, enable flags one at a time
# Priority order (most value first):
# 1. strictNullChecks — catches the most real bugs
# 2. noImplicitAny — forces type annotations
# 3. strictFunctionTypes — catches callback type errors
# 4. noUncheckedIndexedAccess — catches array out-of-bounds

# Step 3: Fix errors per-directory
# Start with shared/domain → application → infrastructure → tests
```

### Prisma Type Errors

```typescript
// Prisma generates strict types — common errors and fixes

// Error: Type '{ name: string }' is not assignable to 'OrderCreateInput'
// Cause: Missing required fields
await prisma.order.create({
  data: { name: 'test' }  // Missing customerId, status, etc.
})
// Fix: Include all required fields
await prisma.order.create({
  data: {
    customerId: customer.id,
    status: 'DRAFT',
    items: { create: [...] }
  }
})

// Error: Type 'string' is not assignable to type 'OrderStatus'
// Cause: Prisma enums are specific types, not strings
await prisma.order.update({
  where: { id },
  data: { status: statusFromApi }  // string, not OrderStatus
})
// Fix: Validate the enum value
import { OrderStatus } from '@prisma/client'
const validStatuses = Object.values(OrderStatus)
if (!validStatuses.includes(statusFromApi as OrderStatus)) {
  throw new ValidationError('Invalid status')
}
await prisma.order.update({
  where: { id },
  data: { status: statusFromApi as OrderStatus }
})

// Error: Property 'items' does not exist on type 'Order'
// Cause: Relations not included in query
const order = await prisma.order.findUnique({ where: { id } })
order.items  // Error — items not loaded
// Fix: Include the relation, or use a type that reflects what's loaded
const order = await prisma.order.findUnique({
  where: { id },
  include: { items: true }
})
// Now TypeScript knows order.items exists
```

---

## 6. Rules for Suppressions

When you MUST suppress a lint rule:

```typescript
// Always use next-line, never file-level disable
// Always include a comment explaining WHY

// Good — justified suppression
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- renderToBuffer types don't match React 19
const buffer = await renderToBuffer(element as any)

// Good — known library limitation
// @ts-expect-error — Chart.js types don't support this config in v4, fix when v5 types land
chart.options.plugins.tooltip.external = customTooltip

// Bad — no justification
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const data: any = response.data

// Bad — file-level disable
/* eslint-disable @typescript-eslint/no-explicit-any */
```

### Suppression Rules

| Do                                                  | Don't                                     |
| --------------------------------------------------- | ----------------------------------------- |
| Use `eslint-disable-next-line`                      | Use `eslint-disable` for whole file       |
| Use `@ts-expect-error` (breaks when error is fixed) | Use `@ts-ignore` (silently hides forever) |
| Include a WHY comment                               | Suppress without explanation              |
| Suppress one specific rule                          | Suppress all rules on a line              |
| Fix the root cause first                            | Jump straight to suppression              |
| Never suppress security rules                       | Suppress `no-unsafe-*` rules              |

---

## 7. Auto-Fix Guide

```bash
# What can be auto-fixed vs what needs manual work

# ESLint — many rules are auto-fixable
npx eslint . --fix
# Auto-fixes: prefer-const, no-extra-semi, import/order, quotes, indent, etc.
# Manual: no-unused-vars, no-explicit-any, exhaustive-deps, security rules

# Prettier — formats everything it can
npx prettier --write .

# Angular lint
ng lint --fix

# TypeScript — no auto-fix (type errors always need manual intervention)
npx tsc --noEmit
# Fix manually, then re-run

# Order of operations:
# 1. npx prettier --write .       (formatting)
# 2. npx eslint . --fix           (auto-fixable lint)
# 3. npx eslint .                 (see remaining manual fixes)
# 4. npx tsc --noEmit             (type errors — always manual)
```

---

## 8. Output

```
## Lint Fix Summary

**Errors fixed:** 12
**Warnings fixed:** 3
**Suppressions added:** 1 (with justification)

### Root causes:
- Missing `OrderStatus` import caused 8 type errors across 4 files
- Nullable `user` parameter wasn't checked before access (3 errors)
- Unused import in 1 file

### Changes:
- `src/services/order.service.ts` — added null check for user before accessing properties
- `src/components/ui/select.tsx` — fixed generic type parameter
- `src/lib/cart.ts` — removed unused import `useState`

### Suppressions:
- `src/lib/pdf.ts:42` — `@ts-expect-error` on renderToBuffer call (React 19 type mismatch)

### Verified:
- ✓ `npx tsc --noEmit` — 0 errors
- ✓ `npm run lint` — 0 errors, 0 warnings
```

## Key Principles

- **"Think of types as sets of values"** (Dan Vanderkam — Effective TypeScript) — a `string | number` is a union of two sets. Understanding this makes type errors intuitive.
- **Use type narrowing over assertions** (Effective TypeScript) — `if (x !== null)` is safer than `x!`. Narrowing is verified by the compiler; assertions are trust-me promises.
- **"Improving quality reduces development costs"** (Steve McConnell — Code Complete) — fixing lint/type errors early prevents cascading bugs later.
- **Types are documentation** — a well-typed function signature tells you more than a comment ever could.

## Inspired By

- **Effective TypeScript** — Dan Vanderkam
- **Programming TypeScript** — Boris Cherny
- **Clean Code** — Robert C. Martin
- **Code Complete** — Steve McConnell
