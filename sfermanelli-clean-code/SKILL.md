---
name: sfermanelli-clean-code
description: Clean up code for readability, consistency, and maintainability without changing behavior. Use this skill whenever the user asks to clean up code, improve readability, make code cleaner, tidy up a file, improve naming, or says things like "this is messy", "clean this up", "make this more readable", "improve code quality", "this file is hard to read". Distinct from refactoring (which changes structure) — this is cosmetic improvements that make code easier to understand.
---

# Clean Code

Make code readable, consistent, and maintainable — without changing what it does. Clean code reads like well-written prose: every name tells you what it does, every function does one thing, every comment adds value.

## Golden Rule

Clean code changes are **behavior-preserving**. The code does exactly the same thing before and after. If you need to change structure or logic, that's refactoring — a different skill.

---

## 1. Meaningful Names

Names are the most powerful readability tool. A good name eliminates the need for a comment.

### Intention-Revealing Names

The name should answer: why does it exist, what does it do, how is it used?

```typescript
// Bad — what is d? what does 86400000 mean?
const d = Date.now() - u.c
if (d > 86400000) { ... }

// Good — the code reads like English
const timeSinceCreation = Date.now() - user.createdAt
if (timeSinceCreation > ONE_DAY_MS) { ... }
```

### Avoid Disinformation

Don't use names that hint at the wrong thing:

```typescript
// Bad — it's not a list, it's a map
const accountList = new Map<string, Account>()

// Good
const accountsById = new Map<string, Account>()
```

### Make Meaningful Distinctions

If names must be different, make the difference meaningful:

```typescript
// Bad — what's the difference?
function getActiveAccount() { ... }
function getActiveAccountInfo() { ... }
function getActiveAccountData() { ... }

// Good — each name tells you what's different
function getAccount() { ... }
function getAccountWithOrders() { ... }
function getAccountSummary() { ... }
```

### Pronounceable Names

If you can't say it in a conversation, rename it:

```typescript
// Bad
const genymdhms = new Date()

// Good
const generatedAt = new Date()
```

### Searchable Names

Single-letter names and magic numbers are impossible to grep:

```typescript
// Bad — try searching for '7' or 'e' in a codebase
for (let i = 0; i < 7; i++) {
  s += t[i] * e
}

// Good
const DAYS_PER_WEEK = 7
for (let day = 0; day < DAYS_PER_WEEK; day++) {
  total += taskEstimate[day] * hoursPerDay
}
```

### Avoid Encodings & Mental Mapping

Don't make the reader translate your code in their head:

```typescript
// Bad — Hungarian notation, type prefixes
const strName: string = ''
const iCount: number = 0
const m_description = ''

// Good — the type system handles this
const name = ''
const count = 0
const description = ''
```

### Naming Convention Summary

| Thing          | Pattern           | Example                                |
| -------------- | ----------------- | -------------------------------------- |
| Functions      | verb + noun       | `calculateFees`, `sendNotification`    |
| Booleans       | is/has/can/should | `isActive`, `hasPermission`, `canEdit` |
| Collections    | plural            | `orders`, `activeStores`               |
| Constants      | SCREAMING_SNAKE   | `MAX_FILE_SIZE`, `PAGE_SIZE`           |
| Components     | PascalCase, noun  | `StoreCard`, `OrderHeader`             |
| Hooks          | use + verb/noun   | `useCart`, `usePagination`             |
| Event handlers | handle + event    | `handleSubmit`, `handleClick`          |

---

## 2. Functions

### Keep Functions Short

A function should fit on one screen (~20-30 lines). If it's longer, it's probably doing more than one thing.

### Do One Thing

A function should do one thing, do it well, and do it only. Test: if you can extract another function with a name that isn't just a restatement of the implementation, it's doing more than one thing.

```typescript
// Bad — doing 3 things
function processOrder(order: Order) {
  if (!order.items.length) throw new Error('Empty')
  const subtotal = order.items.reduce((s, i) => s + i.price * i.qty, 0)
  const total = subtotal + subtotal * 0.1
  await db.orders.insert({ ...order, subtotal, total })
}

// Good — each function does one thing
function validateOrder(order: Order) { ... }
function calculateOrderTotals(items: OrderItem[]) { ... }
function saveOrder(order: Order, totals: OrderTotals) { ... }
```

### One Level of Abstraction

Don't mix high-level and low-level operations:

```typescript
// Bad — mixing levels
function renderPage() {
  const html = getPageHtml() // high level
  html.replace(/&/g, '&amp;') // very low level
  return sendResponse(html) // high level
}

// Good — same level throughout
function renderPage() {
  const html = getPageHtml()
  const sanitized = sanitizeHtml(html)
  return sendResponse(sanitized)
}
```

### Function Arguments — Less Is Better

- **Zero** (niladic) — ideal
- **One** (monadic) — good: `validate(email)`
- **Two** (dyadic) — acceptable: `createPoint(x, y)`
- **Three or more** — use an object:

```typescript
// Bad
function createOrder(storeId, startDate, endDate, items, userId) { ... }

// Good
function createOrder(params: CreateOrderParams) { ... }
```

### No Side Effects

A function named `checkPassword` should NOT also initialize a session. Do what the name says, nothing more.

### Command-Query Separation

A function should either DO something (command) or ANSWER something (query), never both:

```typescript
// Bad — does it set the attribute, or check if it exists?
if (set('username', 'bob')) { ... }

// Good
if (attributeExists('username')) {
  setAttribute('username', 'bob')
}
```

### Prefer Exceptions to Error Codes

Error codes force nested ifs. Exceptions separate the happy path from error handling.

### The Writing Process

Writing clean functions is like writing prose: first draft gets it working, second draft makes it clean. Write it, then polish — extract functions, rename variables, remove duplication. Don't expect perfection on the first pass.

---

## 3. Comments

Comments should explain WHY, not WHAT. If you need a comment to explain what the code does, rewrite the code to be self-explanatory.

### Good Comments

**Explaining intent — WHY this decision was made:**

```typescript
// We use a 60-second window because profiles are created by a DB trigger
// before the OAuth callback completes
const isNewUser = Date.now() - createdAt < 60_000
```

**Explaining context that isn't obvious:**

```typescript
// PostgREST's .or() accepts raw filter strings supporting ::text casting,
// unlike .filter() which doesn't
query = query.or(`id::text.ilike.%${searchTerm}%`)
```

**Warnings about consequences:**

```typescript
// WARNING: Clears ALL sessions globally, not just the current one.
// Used only for admin ban operations.
await service.auth.admin.signOut(userId, 'global')
```

**TODOs with context and owner:**

```typescript
// TODO(seb): Replace with batch API when available — currently 1 req/sec
```

### Bad Comments — Remove These

**Redundant** — restating what the code already says:

```typescript
// Bad: Set the name
const name = user.displayName // delete the comment
```

**Noise** — zero information added:

```typescript
// Bad: Default constructor
constructor() { }
```

**Misleading** — says one thing, code does another

**Commented-out code** — git exists for history

**Changelog/version comments** — that's what git log is for

**Journal comments** — don't maintain a changelog in the file

### The Refactoring Principle

If you feel the urge to write a comment, first try to make the code self-explanatory:

```typescript
// Bad — needs comment
// Check if employee is eligible for benefits
if (employee.flags & HOURLY_FLAG && employee.age > 65) { ... }

// Good — self-documenting
if (employee.isEligibleForBenefits()) { ... }
```

---

## 4. Formatting

Code formatting is about communication. Well-formatted code tells a story top to bottom.

### The Newspaper Metaphor

A source file should read like a newspaper article:

- **Top**: the headline (exports, public API)
- **Middle**: the details (implementation)
- **Bottom**: the lowest-level helpers

### Vertical Density & Openness

Related code should be close together. Separate concepts with blank lines:

```typescript
const supabase = await createClient()

const { data: user } = await supabase.auth.getUser()
if (!user) return { error: 'notAuthenticated' }

const { data: store } = await supabase.from('stores').select('id, name').eq('id', storeId).single()
```

### Horizontal Alignment

Don't over-align — it draws attention to the wrong thing:

```typescript
// Bad
const name = 'John'
const age = 30
const emailAddress = 'john@example.com'

// Good
const name = 'John'
const age = 30
const emailAddress = 'john@example.com'
```

### Placement Rules

- **Imports** at the top, grouped: external → internal → relative
- **Constants** before the function that uses them
- **Types** next to the functions that consume them
- **Helper functions** below the main function

---

## 5. Dead Code Removal

Remove anything not used:

- Unused imports, variables, functions, parameters
- Commented-out code blocks
- Unreachable code after `return`, `throw`, `break`
- Empty catch blocks, empty if branches
- Unused dependencies in `package.json`

---

## 6. Professional Standards

- **Consistency over preference** — follow the project's existing style, not your personal preference
- **Boy Scout Rule** — leave the code cleaner than you found it, but don't rewrite the whole file
- **Small changes** — clean code in small, reviewable chunks. Don't touch 50 files in one pass.
- **Verify** — always run `tsc` and `lint` after cleaning. Clean code that doesn't compile is dirty code.

---

## Output

```
## Clean Code Summary

### Naming (X improvements)
- `d` → `createdDate`, `u` → `user`, `res` → `response`

### Functions (X improvements)
- Reduced 80-line function to 25 lines by extracting helpers
- Converted 5-arg function to object parameter

### Comments (X changes)
- Removed 8 redundant comments
- Added 2 intent comments on non-obvious logic
- Removed 15 lines of commented-out code

### Formatting (X changes)
- Added blank lines between logical sections
- Reordered functions to follow newspaper flow

### Dead Code (X removals)
- Removed 4 unused imports, 2 unused variables

### Verified: behavior unchanged
- ✓ `npx tsc --noEmit` passes
- ✓ `npm run lint` passes
```

## Inspired By

- **Clean Code** — Robert C. Martin
- **A Philosophy of Software Design** — John Ousterhout
- **The Art of Readable Code** — Dustin Boswell & Trevor Foucher
- **Code Complete** — Steve McConnell
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
