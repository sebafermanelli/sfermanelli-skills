---
name: sfermanelli-refactor
description: Restructure code to improve architecture without changing behavior. Use this skill when the user asks to refactor, restructure, reorganize, extract components, split files, move logic, reduce duplication, or improve architecture. Triggers for "this file is too big", "extract this into a component", "split this", "DRY this up", "reduce coupling", "improve the architecture", "this is getting complex". Different from clean-code (cosmetic) — this changes code structure.
---

# Refactor

Restructure code to be better organized, more maintainable, and easier to extend — without changing what it does.

> "Refactoring is a disciplined technique for restructuring an existing body of code, altering its internal structure without changing its external behavior." — Martin Fowler

## Golden Rule

**Refactoring changes structure, not behavior.** If the app works differently after your refactoring, you introduced a bug. Every refactoring step should be verifiable: run the tests, run the type checker, manually verify the feature still works.

---

## 1. When to Refactor

### Code Smells as Signals

| Smell                      | Signal                                           | Threshold                         |
| -------------------------- | ------------------------------------------------ | --------------------------------- |
| **Long Method**            | Scrolling to read a function                     | > 30 lines                        |
| **Large Class/File**       | File does multiple unrelated things              | > 300 lines                       |
| **Feature Envy**           | Method uses more data from another class         | > 3 accesses to another object    |
| **Data Clumps**            | Same group of fields appear together             | Same 3+ params in 3+ places       |
| **Primitive Obsession**    | Using strings/numbers for domain concepts        | Email as string, money as number  |
| **Long Parameter List**    | Function signature is hard to read               | > 3 parameters                    |
| **Divergent Change**       | One class changed for multiple unrelated reasons | Changed for 3+ different features |
| **Shotgun Surgery**        | One change requires editing many files           | > 5 files touched for one feature |
| **Duplicated Code**        | Same logic in 3+ places                          | Rule of Three                     |
| **Speculative Generality** | Abstractions with only one implementation        | Unused interfaces, unused params  |

### When NOT to Refactor

- **Before a deadline** — refactoring takes time and can introduce bugs
- **Without tests** — you need a safety net. Write characterization tests first.
- **Without understanding the code** — read first, refactor second
- **While debugging** — fix the bug, then refactor in a separate commit
- **For aesthetic reasons only** — if it's not causing problems, leave it alone

---

## 2. Refactoring Patterns (Martin Fowler)

### Extract Method

When a function does too much:

```typescript
// Before — 60-line function
async function processOrder(order: Order) {
  // 15 lines of validation
  if (!order.items.length) throw new Error('Empty')
  if (order.total <= 0) throw new Error('Invalid total')
  for (const item of order.items) {
    if (item.quantity <= 0) throw new Error('Invalid quantity')
  }

  // 15 lines of pricing
  const subtotal = order.items.reduce((s, i) => s + i.price * i.quantity, 0)
  const tax = subtotal * 0.21
  const discount = order.coupon ? subtotal * order.coupon.rate : 0
  const total = subtotal + tax - discount

  // 15 lines of persistence
  await db.orders.insert({ ...order, subtotal, tax, discount, total })
  await db.inventory.decrementMany(order.items)

  // 15 lines of notifications
  await sendOrderConfirmation(order.customerId, total)
  await notifyWarehouse(order.id, order.items)
}

// After — each function does one thing
async function processOrder(order: Order) {
  validateOrder(order)
  const totals = calculateTotals(order)
  await saveOrder(order, totals)
  await sendNotifications(order, totals)
}

function validateOrder(order: Order): void {
  if (!order.items.length) throw new EmptyOrderError(order.id)
  if (order.items.some(i => i.quantity <= 0)) throw new InvalidQuantityError()
}

function calculateTotals(order: Order): OrderTotals {
  const subtotal = order.items.reduce((s, i) => s + i.price * i.quantity, 0)
  const tax = subtotal * 0.21
  const discount = order.coupon ? subtotal * order.coupon.rate : 0
  return { subtotal, tax, discount, total: subtotal + tax - discount }
}
```

### Extract Class

When a class has too many responsibilities:

```typescript
// Before — OrderService does CRUD + pricing + notifications + analytics
class OrderService {
  async create(input: CreateOrderInput) { ... }     // CRUD
  async update(id: string, input: UpdateInput) { ... }
  async delete(id: string) { ... }
  calculateSubtotal(items: OrderItem[]) { ... }     // Pricing
  calculateTax(subtotal: number, region: string) { ... }
  applyDiscount(total: number, coupon: Coupon) { ... }
  sendConfirmation(order: Order) { ... }             // Notifications
  sendShippingUpdate(order: Order) { ... }
  trackOrderCreated(order: Order) { ... }            // Analytics
  trackOrderCompleted(order: Order) { ... }
}

// After — split by responsibility
class OrderService {                        // Only CRUD + orchestration
  constructor(
    private pricing: PricingService,
    private notifications: NotificationService,
    private analytics: AnalyticsService
  ) {}

  async create(input: CreateOrderInput) {
    const totals = this.pricing.calculate(input.items, input.coupon)
    const order = await this.repository.save({ ...input, ...totals })
    await this.notifications.sendConfirmation(order)
    this.analytics.trackCreated(order)
    return order
  }
}

class PricingService {                       // Only pricing
  calculate(items: OrderItem[], coupon?: Coupon): OrderTotals { ... }
}

class NotificationService {                  // Only notifications
  sendConfirmation(order: Order): Promise<void> { ... }
  sendShippingUpdate(order: Order): Promise<void> { ... }
}
```

### Replace Conditional with Polymorphism

When the same switch/if appears in multiple places:

```typescript
// Before — switch repeated in 3+ places
function calculateShipping(order: Order): number {
  switch (order.shippingMethod) {
    case 'standard':
      return order.weight * 0.5
    case 'express':
      return order.weight * 1.5 + 10
    case 'overnight':
      return order.weight * 3 + 25
    case 'free':
      return 0
  }
}

function getEstimatedDelivery(order: Order): number {
  switch (order.shippingMethod) {
    case 'standard':
      return 7
    case 'express':
      return 3
    case 'overnight':
      return 1
    case 'free':
      return 14
  }
}

function getTrackingUrl(order: Order): string | null {
  switch (order.shippingMethod) {
    case 'standard':
      return null
    case 'express':
      return `https://track.express.com/${order.trackingId}`
    case 'overnight':
      return `https://track.overnight.com/${order.trackingId}`
    case 'free':
      return null
  }
}

// After — polymorphism (each method is responsible for its own behavior)
interface ShippingStrategy {
  calculateCost(weight: number): Money
  estimatedDays(): number
  trackingUrl(trackingId: string): string | null
}

class StandardShipping implements ShippingStrategy {
  calculateCost(weight: number) {
    return Money.fromUnits(weight * 0.5, 'USD')
  }
  estimatedDays() {
    return 7
  }
  trackingUrl() {
    return null
  }
}

class ExpressShipping implements ShippingStrategy {
  calculateCost(weight: number) {
    return Money.fromUnits(weight * 1.5 + 10, 'USD')
  }
  estimatedDays() {
    return 3
  }
  trackingUrl(trackingId: string) {
    return `https://track.express.com/${trackingId}`
  }
}

class OvernightShipping implements ShippingStrategy {
  calculateCost(weight: number) {
    return Money.fromUnits(weight * 3 + 25, 'USD')
  }
  estimatedDays() {
    return 1
  }
  trackingUrl(trackingId: string) {
    return `https://track.overnight.com/${trackingId}`
  }
}

// Factory to get the right strategy
function getShippingStrategy(method: ShippingMethod): ShippingStrategy {
  const strategies: Record<ShippingMethod, ShippingStrategy> = {
    standard: new StandardShipping(),
    express: new ExpressShipping(),
    overnight: new OvernightShipping(),
    free: new FreeShipping(),
  }
  return strategies[method]
}

// Usage — no more switch statements
const shipping = getShippingStrategy(order.shippingMethod)
const cost = shipping.calculateCost(order.weight)
const days = shipping.estimatedDays()
const url = shipping.trackingUrl(order.trackingId)
```

### Introduce Parameter Object

When the same group of parameters appears together:

```typescript
// Before — same 4 params in multiple functions
function searchOrders(
  storeId: string, status: string, startDate: Date, endDate: Date, page: number, pageSize: number
) { ... }

function exportOrders(
  storeId: string, status: string, startDate: Date, endDate: Date, format: string
) { ... }

function countOrders(
  storeId: string, status: string, startDate: Date, endDate: Date
) { ... }

// After — parameter object
interface OrderFilters {
  storeId: string
  status?: OrderStatus
  dateRange?: { start: Date; end: Date }
}

function searchOrders(filters: OrderFilters, pagination: Pagination) { ... }
function exportOrders(filters: OrderFilters, format: ExportFormat) { ... }
function countOrders(filters: OrderFilters) { ... }
```

### Replace Primitive with Value Object

When a primitive carries domain meaning:

```typescript
// Before — primitives everywhere
function createOrder(
  customerId: string,     // Which string is which?
  storeId: string,
  email: string,
  total: number,          // Cents? Dollars? What currency?
  currency: string
) { ... }

// Bug: easy to swap customerId and storeId — both are strings
createOrder(storeId, customerId, email, total, currency)  // WRONG but compiles

// After — value objects
function createOrder(
  customerId: CustomerId,
  storeId: StoreId,
  email: Email,
  total: Money
) { ... }

// Can't swap — different types
createOrder(storeId, customerId, email, total)  // Compile error!

// Value objects also encapsulate validation
class Email {
  readonly value: string
  constructor(value: string) {
    const normalized = value.trim().toLowerCase()
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(normalized)) {
      throw new InvalidEmailError(value)
    }
    this.value = normalized
  }
}
```

### Extract Component (React/Angular)

When a UI component does too much:

```typescript
// Before: 200-line component with inline form
export function StoreSettings({ store }: { store: Store }) {
  // ...50 lines of store info display
  // ...50 lines of form state
  // ...50 lines of form handlers
  // ...50 lines of form JSX
}

// After: composed from focused components
export function StoreSettings({ store }: { store: Store }) {
  return (
    <div>
      <StoreInfo store={store} />
      <StoreSettingsForm store={store} onSaved={handleSaved} />
    </div>
  )
}
```

### Extract Hook (React)

When component state logic is complex or reusable:

```typescript
// Before: 30 lines of pagination logic in component
function OrderList() {
  const [page, setPage] = useState(1)
  const [pageSize] = useState(20)
  const [total, setTotal] = useState(0)
  const [data, setData] = useState<Order[]>([])

  useEffect(() => {
    fetchOrders({ page, pageSize }).then(({ data, total }) => {
      setData(data)
      setTotal(total)
    })
  }, [page, pageSize])

  const totalPages = Math.ceil(total / pageSize)
  // ...more pagination logic
}

// After: custom hook
function usePagination<T>(fetcher: (params: PaginationParams) => Promise<PaginatedResult<T>>) {
  const [page, setPage] = useState(1)
  const [pageSize] = useState(20)
  const [result, setResult] = useState<PaginatedResult<T>>({ data: [], total: 0 })

  useEffect(() => {
    fetcher({ page, pageSize }).then(setResult)
  }, [page, pageSize, fetcher])

  return {
    data: result.data,
    page,
    totalPages: Math.ceil(result.total / pageSize),
    nextPage: () => setPage(p => Math.min(p + 1, Math.ceil(result.total / pageSize))),
    prevPage: () => setPage(p => Math.max(p - 1, 1)),
    goToPage: setPage,
  }
}

// Usage
function OrderList() {
  const { data, page, totalPages, nextPage, prevPage } = usePagination(fetchOrders)
  // ...render
}
```

### Split by Concern

When a file does too many things:

```
// Before
actions/stores.ts (800 lines: CRUD + geocoding + schedules + fees)

// After
actions/stores.ts (create, update, delete — 200 lines)
actions/store-schedules.ts (schedule CRUD — 150 lines)
actions/store-fees.ts (fee calculations — 200 lines)
services/geocoding.ts (address → coordinates — 100 lines)
```

---

## 3. Simplify Conditionals

### Early Returns / Guard Clauses

```typescript
// Before: deeply nested
function processOrder(order: Order, user: User) {
  if (user) {
    if (user.isActive) {
      if (order.items.length > 0) {
        if (order.total > 0) {
          // actual logic buried 4 levels deep
          return saveOrder(order)
        } else {
          throw new Error('Invalid total')
        }
      } else {
        throw new Error('Empty order')
      }
    } else {
      throw new Error('Inactive user')
    }
  } else {
    throw new Error('No user')
  }
}

// After: flat with guard clauses
function processOrder(order: Order, user: User) {
  if (!user) throw new UnauthorizedError()
  if (!user.isActive) throw new InactiveUserError(user.id)
  if (order.items.length === 0) throw new EmptyOrderError(order.id)
  if (order.total <= 0) throw new InvalidTotalError(order.total)

  return saveOrder(order)
}
```

### Replace Nested Conditional with Strategy Map

```typescript
// Before: long if-else chain
function getStatusBadge(status: OrderStatus) {
  if (status === 'draft') return { color: 'gray', label: 'Draft' }
  else if (status === 'placed') return { color: 'blue', label: 'Placed' }
  else if (status === 'processing') return { color: 'yellow', label: 'Processing' }
  else if (status === 'shipped') return { color: 'purple', label: 'Shipped' }
  else if (status === 'delivered') return { color: 'green', label: 'Delivered' }
  else if (status === 'cancelled') return { color: 'red', label: 'Cancelled' }
  else return { color: 'gray', label: 'Unknown' }
}

// After: strategy map
const STATUS_BADGES: Record<OrderStatus, { color: string; label: string }> = {
  draft: { color: 'gray', label: 'Draft' },
  placed: { color: 'blue', label: 'Placed' },
  processing: { color: 'yellow', label: 'Processing' },
  shipped: { color: 'purple', label: 'Shipped' },
  delivered: { color: 'green', label: 'Delivered' },
  cancelled: { color: 'red', label: 'Cancelled' },
}

function getStatusBadge(status: OrderStatus) {
  return STATUS_BADGES[status] ?? { color: 'gray', label: 'Unknown' }
}
```

---

## 4. Metrics and Indicators

### When Refactoring Is Needed (Measurable)

| Metric                    | Tool                              | Red Flag                 |
| ------------------------- | --------------------------------- | ------------------------ | --------------------------- |
| **Cyclomatic complexity** | ESLint `complexity` rule          | > 10 per function        |
| **File length**           | `wc -l`                           | > 300 lines              |
| **Function length**       | ESLint `max-lines-per-function`   | > 30 lines               |
| **Parameter count**       | ESLint `max-params`               | > 3 parameters           |
| **Import count**          | Count imports                     | > 15 imports in one file |
| **Depth of nesting**      | ESLint `max-depth`                | > 3 levels               |
| **Churn rate**            | `git log --format='%H' -- file.ts | wc -l`                   | File changed in 20+ commits |

---

## 5. Process

1. **Identify the smell** — what's wrong with the current structure?
2. **Ensure tests exist** — if not, write characterization tests first
3. **Plan the change** — describe what moves where
4. **Make one change at a time** — don't do 5 refactorings in one step
5. **Verify after each step** — `npx tsc --noEmit`, run tests
6. **Update imports** — make sure nothing points to the old location
7. **Commit each step separately** — so you can revert one step without losing others

---

## 6. Anti-Patterns

- **Premature abstraction** — don't abstract until you have 3+ concrete cases ("Rule of Three")
- **Over-engineering** — a simple function doesn't need a class, interface, factory, and strategy pattern
- **Refactoring under pressure** — don't refactor while fixing a production bug
- **Big bang refactoring** — prefer small, incremental changes over rewriting everything at once
- **Moving code without understanding it** — read first, refactor second
- **DRY as dogma** — three similar lines are sometimes better than a premature abstraction. "Duplication is far cheaper than the wrong abstraction." — Sandi Metz

---

## 7. Output

```markdown
## Refactoring Summary

**Goal:** [what structural problem was solved]
**Pattern:** [which refactoring pattern was applied]

### Changes:

1. Extracted `PricingService` from `OrderService` (was 600 lines, now 300 + 200)
2. Replaced shipping switch with Strategy pattern (eliminated 3 duplicated switches)
3. Introduced `Money` value object (replaced raw number + currency string in 12 places)

### Files:

- `src/services/order.service.ts` — extracted pricing and notifications
- `src/services/pricing.service.ts` — new, extracted
- `src/domain/shipping-strategy.ts` — new, replaces switch statements
- `src/domain/money.ts` — new value object

### Verified:

- ✓ `npx tsc --noEmit` — 0 errors
- ✓ All tests pass (42/42)
- ✓ No import changes needed in consuming files
```

## Key Principles

### Deep Modules (Ousterhout)

A good module has a **simple interface** hiding **complex implementation**. If your refactoring makes the interface more complex than what it hides, you're going the wrong direction.

### Working with Legacy Code (Feathers)

When refactoring untested code:

- **Characterize first** — write tests that capture current behavior before changing anything
- **Sprout Method** — write new functionality in a new method, call it from the old code
- **Wrap Method** — create a new method with the old name, rename the old one

> "Legacy code is code without tests." — Michael Feathers

### The Wrong Abstraction (Sandi Metz)

> "Prefer duplication over the wrong abstraction. Duplication is far cheaper than the wrong abstraction."

If an abstraction requires passing boolean flags or configuration to handle different cases, it might be the wrong abstraction. Consider inlining it back and letting the duplicated code reveal the right pattern.

## Inspired By

- **Refactoring** — Martin Fowler
- **A Philosophy of Software Design** — John Ousterhout
- **Working Effectively with Legacy Code** — Michael Feathers
- **Clean Code** — Robert C. Martin
- **refactoring.guru** — Alexander Shvets
