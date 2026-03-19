---
name: sfermanelli-write-tests
description: Generate well-structured, production-quality tests for any codebase. Use this skill whenever the user asks to write tests, add test coverage, create unit/integration/e2e tests, test a function/component/API route, or mentions testing in any form — even casually like "this needs tests" or "can you test this". Also triggers for "add coverage", "write specs", "test this flow", or "make sure this works".
---

# Write Tests

Generate clean, maintainable, production-quality tests that actually catch bugs — not tests that just exist to inflate coverage numbers.

## Golden Rule

**Tests should break when the code is wrong, not when the code is refactored.** Test behavior, not implementation. If you rename a private method and 10 tests break, those tests are coupled to implementation details.

---

## 1. Philosophy

Good tests have three properties:

1. **They break when the code is wrong** — if you can change the implementation to be buggy and the test still passes, the test is useless
2. **They don't break when the code is right** — brittle tests that fail on every refactor waste more time than they save
3. **They document behavior** — reading the test should tell you what the code does without reading the implementation

> "The first rule of testing: tests should tell you when something breaks. The second rule: tests should not tell you when nothing broke." — Kent Beck

---

## 2. Before Writing Tests

1. **Detect the project's test stack** — check `package.json`, `vitest.config.*`, `jest.config.*`, `playwright.config.*`, existing test files. Don't assume — detect.
2. **Find existing test patterns** — look at `**/*.test.*`, `**/*.spec.*`, `__tests__/` directories. Match the project's conventions.
3. **Understand what you're testing** — read the source code. Identify inputs, outputs, side effects, error paths, and edge cases.

---

## 3. Test Structure

### Arrange-Act-Assert

Every test should be readable top-to-bottom:

```typescript
it('applies 20% discount when valid coupon is used', () => {
  // Arrange — set up the test data
  const product = createProduct({ price: Money.fromUnits(100, 'USD') })
  const coupon = createCoupon({ discountPercent: 20 })

  // Act — execute the behavior under test
  const result = applyDiscount(product, coupon)

  // Assert — verify the outcome
  expect(result.finalPrice.cents).toBe(8000)
})
```

### Naming Convention

Test names should describe **behavior**, not implementation:

```typescript
// Bad — describes implementation
it('calls calculateFee')
it('returns true')
it('sets state to loading')

// Good — describes behavior
it('charges a 10% platform fee on the subtotal')
it('allows checkout when all items are in stock')
it('shows loading spinner while fetching orders')
```

Use `describe` blocks to group by feature or scenario:

```typescript
describe('OrderService.place', () => {
  it('creates an order with the correct total')
  it('reserves stock for each item')
  it('emits an OrderPlaced event')

  describe('when stock is insufficient', () => {
    it('returns an InsufficientStockError')
    it('does not create the order')
    it('does not charge the customer')
  })

  describe('when payment fails', () => {
    it('releases the reserved stock')
    it('returns a PaymentFailedError')
  })
})
```

---

## 4. Test Factories and Fixtures

### Object Factories

Create reusable factories for test data. Avoid duplicating object creation across tests.

```typescript
// test/factories/order.factory.ts
import { faker } from '@faker-js/faker'

interface OrderOverrides {
  id?: string
  status?: OrderStatus
  customerId?: string
  items?: OrderItem[]
  total?: number
}

export function createOrder(overrides: OrderOverrides = {}): Order {
  return {
    id: overrides.id ?? faker.string.uuid(),
    status: overrides.status ?? 'DRAFT',
    customerId: overrides.customerId ?? faker.string.uuid(),
    items: overrides.items ?? [createOrderItem()],
    total: overrides.total ?? 4999,
    createdAt: new Date(),
    updatedAt: new Date(),
  }
}

export function createOrderItem(overrides: Partial<OrderItem> = {}): OrderItem {
  return {
    id: overrides.id ?? faker.string.uuid(),
    productId: overrides.productId ?? faker.string.uuid(),
    productName: overrides.productName ?? faker.commerce.productName(),
    quantity: overrides.quantity ?? 1,
    priceInCents: overrides.priceInCents ?? faker.number.int({ min: 100, max: 10000 }),
  }
}

// Usage — only specify what matters for this test
it('calculates total from item prices', () => {
  const order = createOrder({
    items: [
      createOrderItem({ priceInCents: 1000, quantity: 2 }),
      createOrderItem({ priceInCents: 500, quantity: 1 }),
    ],
  })
  expect(calculateTotal(order)).toBe(2500)
})
```

### Database Fixtures (Integration Tests)

```typescript
// test/fixtures/database.ts
export async function seedTestData(prisma: PrismaClient) {
  const customer = await prisma.customer.create({
    data: {
      id: 'test-customer-1',
      name: 'Test Customer',
      email: 'test@example.com',
    },
  })

  const store = await prisma.store.create({
    data: {
      id: 'test-store-1',
      name: 'Test Store',
      ownerId: customer.id,
    },
  })

  return { customer, store }
}

// Clean up between tests
export async function cleanDatabase(prisma: PrismaClient) {
  // Delete in reverse dependency order
  await prisma.orderItem.deleteMany()
  await prisma.order.deleteMany()
  await prisma.store.deleteMany()
  await prisma.customer.deleteMany()
}

// Usage in tests
describe('OrderService (integration)', () => {
  let fixtures: Awaited<ReturnType<typeof seedTestData>>

  beforeEach(async () => {
    await cleanDatabase(prisma)
    fixtures = await seedTestData(prisma)
  })

  it('creates an order for the store', async () => {
    const order = await orderService.create({
      storeId: fixtures.store.id,
      customerId: fixtures.customer.id,
      items: [{ productId: 'prod-1', quantity: 2 }],
    })

    expect(order.storeId).toBe(fixtures.store.id)

    // Verify in database
    const saved = await prisma.order.findUnique({ where: { id: order.id } })
    expect(saved).not.toBeNull()
    expect(saved!.status).toBe('DRAFT')
  })
})
```

---

## 5. What to Test

### Unit Tests (functions, utils, business logic)

```typescript
describe('Money', () => {
  it('adds two amounts in the same currency', () => {
    const a = Money.fromCents(1000, 'USD')
    const b = Money.fromCents(2050, 'USD')
    expect(a.add(b).cents).toBe(3050)
  })

  it('throws when adding different currencies', () => {
    const usd = Money.fromCents(1000, 'USD')
    const eur = Money.fromCents(1000, 'EUR')
    expect(() => usd.add(eur)).toThrow('Cannot operate on USD and EUR')
  })

  it('formats as currency string', () => {
    expect(Money.fromCents(4999, 'USD').format()).toBe('$49.99')
    expect(Money.fromCents(0, 'USD').format()).toBe('$0.00')
  })

  // Edge cases
  it('rejects negative amounts', () => {
    expect(() => Money.fromCents(-100, 'USD')).toThrow()
  })

  it('rounds to nearest cent', () => {
    expect(Money.fromUnits(49.999, 'USD').cents).toBe(5000)
  })
})
```

### Integration Tests (API routes, services with DB)

```typescript
describe('POST /api/orders', () => {
  it('creates an order and returns 201', async () => {
    const response = await request(app)
      .post('/api/orders')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        items: [{ productId: 'prod-1', quantity: 2 }],
        shippingAddress: testAddress,
      })

    expect(response.status).toBe(201)
    expect(response.body.data.id).toBeDefined()
    expect(response.body.data.status).toBe('DRAFT')
  })

  it('returns 400 when items array is empty', async () => {
    const response = await request(app)
      .post('/api/orders')
      .set('Authorization', `Bearer ${testToken}`)
      .send({ items: [], shippingAddress: testAddress })

    expect(response.status).toBe(400)
    expect(response.body.error.code).toBe('VALIDATION_ERROR')
    expect(response.body.error.details[0].field).toBe('items')
  })

  it('returns 401 without auth token', async () => {
    const response = await request(app)
      .post('/api/orders')
      .send({ items: [{ productId: 'prod-1', quantity: 1 }] })

    expect(response.status).toBe(401)
  })

  it('returns 409 when stock is insufficient', async () => {
    // Arrange — product with only 1 in stock
    await prisma.product.update({
      where: { id: 'prod-1' },
      data: { stock: 1 },
    })

    const response = await request(app)
      .post('/api/orders')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        items: [{ productId: 'prod-1', quantity: 5 }],
        shippingAddress: testAddress,
      })

    expect(response.status).toBe(409)
    expect(response.body.error.code).toBe('INSUFFICIENT_STOCK')
  })
})
```

### Component Tests (React)

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

describe('CheckoutForm', () => {
  it('disables submit button when form is incomplete', () => {
    render(<CheckoutForm />)
    expect(screen.getByRole('button', { name: /confirm/i })).toBeDisabled()
  })

  it('enables submit after filling required fields', async () => {
    const user = userEvent.setup()
    render(<CheckoutForm />)

    await user.type(screen.getByLabelText(/email/i), 'test@example.com')
    await user.type(screen.getByLabelText(/name/i), 'John Doe')

    expect(screen.getByRole('button', { name: /confirm/i })).toBeEnabled()
  })

  it('shows validation error for invalid email', async () => {
    const user = userEvent.setup()
    render(<CheckoutForm />)

    await user.type(screen.getByLabelText(/email/i), 'not-an-email')
    await user.tab()  // Trigger blur validation

    expect(screen.getByText(/valid email/i)).toBeInTheDocument()
  })

  it('shows loading state during submission', async () => {
    const user = userEvent.setup()
    render(<CheckoutForm onSubmit={vi.fn(() => new Promise(() => {}))} />)

    await user.type(screen.getByLabelText(/email/i), 'test@example.com')
    await user.type(screen.getByLabelText(/name/i), 'John Doe')
    await user.click(screen.getByRole('button', { name: /confirm/i }))

    expect(screen.getByRole('button', { name: /submitting/i })).toBeDisabled()
  })

  it('displays error message when submission fails', async () => {
    const user = userEvent.setup()
    const onSubmit = vi.fn().mockRejectedValue(new Error('Network error'))
    render(<CheckoutForm onSubmit={onSubmit} />)

    await user.type(screen.getByLabelText(/email/i), 'test@example.com')
    await user.type(screen.getByLabelText(/name/i), 'John Doe')
    await user.click(screen.getByRole('button', { name: /confirm/i }))

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/something went wrong/i)
    })
  })
})
```

### E2E Tests (Playwright)

```typescript
import { test, expect } from '@playwright/test'

test.describe('Order flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login via API to skip UI login
    const token = await getTestAuthToken()
    await page.goto('/')
    await page.evaluate(t => localStorage.setItem('token', t), token)
  })

  test('user can create and place an order', async ({ page }) => {
    // Navigate to store
    await page.goto('/stores/test-store')

    // Add product to cart
    await page
      .getByRole('button', { name: /add to cart/i })
      .first()
      .click()
    await expect(page.getByText(/1 item in cart/i)).toBeVisible()

    // Go to checkout
    await page.getByRole('link', { name: /checkout/i }).click()

    // Fill shipping address
    await page.getByLabel(/street/i).fill('123 Main St')
    await page.getByLabel(/city/i).fill('Springfield')
    await page.getByLabel(/zip/i).fill('62701')

    // Place order
    await page.getByRole('button', { name: /place order/i }).click()

    // Verify confirmation
    await expect(page.getByText(/order confirmed/i)).toBeVisible()
    await expect(page.getByText(/order #/i)).toBeVisible()
  })

  test('shows error when trying to order out-of-stock item', async ({ page }) => {
    await page.goto('/stores/test-store/products/out-of-stock-product')
    await expect(page.getByRole('button', { name: /add to cart/i })).toBeDisabled()
    await expect(page.getByText(/out of stock/i)).toBeVisible()
  })
})
```

### Playwright Page Objects

```typescript
// test/pages/checkout.page.ts
export class CheckoutPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/checkout')
  }

  async fillShippingAddress(address: { street: string; city: string; zip: string }) {
    await this.page.getByLabel(/street/i).fill(address.street)
    await this.page.getByLabel(/city/i).fill(address.city)
    await this.page.getByLabel(/zip/i).fill(address.zip)
  }

  async placeOrder() {
    await this.page.getByRole('button', { name: /place order/i }).click()
  }

  async expectConfirmation() {
    await expect(this.page.getByText(/order confirmed/i)).toBeVisible()
  }

  async expectError(message: string) {
    await expect(this.page.getByRole('alert')).toContainText(message)
  }
}

// Usage — cleaner tests
test('complete checkout flow', async ({ page }) => {
  const checkout = new CheckoutPage(page)
  await checkout.goto()
  await checkout.fillShippingAddress({ street: '123 Main', city: 'Springfield', zip: '62701' })
  await checkout.placeOrder()
  await checkout.expectConfirmation()
})
```

---

## 6. NestJS Testing

### Unit Testing Services

```typescript
import { Test, TestingModule } from '@nestjs/testing'

describe('OrderService', () => {
  let service: OrderService
  let orderRepo: jest.Mocked<OrderRepository>
  let eventBus: jest.Mocked<EventBus>

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderService,
        {
          provide: 'OrderRepository',
          useValue: {
            findById: jest.fn(),
            save: jest.fn(),
            delete: jest.fn(),
          },
        },
        {
          provide: EventBus,
          useValue: {
            publish: jest.fn(),
            publishAll: jest.fn(),
          },
        },
      ],
    }).compile()

    service = module.get<OrderService>(OrderService)
    orderRepo = module.get('OrderRepository')
    eventBus = module.get(EventBus)
  })

  describe('place', () => {
    it('places a draft order and emits event', async () => {
      const order = createOrder({ status: 'DRAFT' })
      orderRepo.findById.mockResolvedValue(order)
      orderRepo.save.mockResolvedValue(undefined)

      await service.place(order.id)

      expect(orderRepo.save).toHaveBeenCalledWith(expect.objectContaining({ status: 'PLACED' }))
      expect(eventBus.publishAll).toHaveBeenCalledWith(
        expect.arrayContaining([expect.objectContaining({ eventType: 'order.placed' })])
      )
    })

    it('throws when order is not found', async () => {
      orderRepo.findById.mockResolvedValue(null)

      await expect(service.place('non-existent')).rejects.toThrow(OrderNotFoundError)
    })

    it('throws when order is already placed', async () => {
      const order = createOrder({ status: 'PLACED' })
      orderRepo.findById.mockResolvedValue(order)

      await expect(service.place(order.id)).rejects.toThrow(OrderNotModifiableError)
    })
  })
})
```

### E2E Testing Controllers

```typescript
import { Test, TestingModule } from '@nestjs/testing'
import { INestApplication, ValidationPipe } from '@nestjs/common'
import * as request from 'supertest'

describe('OrderController (e2e)', () => {
  let app: INestApplication

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // Import the real module
    })
      .overrideProvider('PaymentGateway')
      .useValue({
        charge: jest.fn().mockResolvedValue({ id: 'ch_test', status: 'succeeded' }),
      })
      .compile()

    app = module.createNestApplication()
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))
    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  it('POST /orders — creates order', async () => {
    const response = await request(app.getHttpServer())
      .post('/orders')
      .set('Authorization', `Bearer ${testToken}`)
      .send({ items: [{ productId: 'prod-1', quantity: 2 }] })

    expect(response.status).toBe(201)
    expect(response.body.data.id).toBeDefined()
  })
})
```

---

## 7. Angular Testing

### Component Testing with TestBed

```typescript
import { ComponentFixture, TestBed, fakeAsync, tick } from '@angular/core/testing'
import { By } from '@angular/platform-browser'
import { of, throwError } from 'rxjs'

describe('OrderListComponent', () => {
  let component: OrderListComponent
  let fixture: ComponentFixture<OrderListComponent>
  let orderService: jasmine.SpyObj<OrderService>

  beforeEach(async () => {
    const orderServiceSpy = jasmine.createSpyObj('OrderService', ['getOrders', 'cancelOrder'])

    await TestBed.configureTestingModule({
      imports: [OrderListComponent], // Standalone component
      providers: [{ provide: OrderService, useValue: orderServiceSpy }],
    }).compileComponents()

    fixture = TestBed.createComponent(OrderListComponent)
    component = fixture.componentInstance
    orderService = TestBed.inject(OrderService) as jasmine.SpyObj<OrderService>
  })

  it('displays orders when data loads', () => {
    orderService.getOrders.and.returnValue(
      of([createOrder({ id: '1', status: 'PLACED' }), createOrder({ id: '2', status: 'SHIPPED' })])
    )

    fixture.detectChanges()

    const rows = fixture.debugElement.queryAll(By.css('[data-testid="order-row"]'))
    expect(rows.length).toBe(2)
  })

  it('shows empty state when no orders', () => {
    orderService.getOrders.and.returnValue(of([]))
    fixture.detectChanges()

    const emptyState = fixture.debugElement.query(By.css('[data-testid="empty-state"]'))
    expect(emptyState).toBeTruthy()
    expect(emptyState.nativeElement.textContent).toContain('No orders yet')
  })

  it('shows error state when fetch fails', () => {
    orderService.getOrders.and.returnValue(throwError(() => new Error('Network error')))
    fixture.detectChanges()

    const error = fixture.debugElement.query(By.css('[role="alert"]'))
    expect(error.nativeElement.textContent).toContain('Failed to load orders')
  })

  it('cancels order and refreshes list', fakeAsync(() => {
    const orders = [createOrder({ id: '1', status: 'PLACED' })]
    orderService.getOrders.and.returnValue(of(orders))
    orderService.cancelOrder.and.returnValue(of(undefined))
    fixture.detectChanges()

    const cancelBtn = fixture.debugElement.query(By.css('[data-testid="cancel-order-1"]'))
    cancelBtn.triggerEventHandler('click', null)
    tick()

    expect(orderService.cancelOrder).toHaveBeenCalledWith('1')
    expect(orderService.getOrders).toHaveBeenCalledTimes(2) // Initial + refresh
  }))
})
```

### Service Testing

```typescript
import { TestBed } from '@angular/core/testing'
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing'

describe('OrderService', () => {
  let service: OrderService
  let httpMock: HttpTestingController

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [OrderService],
    })

    service = TestBed.inject(OrderService)
    httpMock = TestBed.inject(HttpTestingController)
  })

  afterEach(() => {
    httpMock.verify() // Ensure no outstanding requests
  })

  it('fetches orders from API', () => {
    const mockOrders = [createOrder({ id: '1' }), createOrder({ id: '2' })]

    service.getOrders().subscribe(orders => {
      expect(orders.length).toBe(2)
      expect(orders[0].id).toBe('1')
    })

    const req = httpMock.expectOne('/api/orders')
    expect(req.request.method).toBe('GET')
    req.flush({ data: mockOrders })
  })

  it('handles error response', () => {
    service.getOrders().subscribe({
      error: err => {
        expect(err.message).toContain('Failed to load orders')
      },
    })

    const req = httpMock.expectOne('/api/orders')
    req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' })
  })
})
```

---

## 8. Mocking Guidelines

### What to Mock

| Mock                                    | Don't Mock                    |
| --------------------------------------- | ----------------------------- |
| External APIs (Stripe, SendGrid)        | The code under test           |
| Database in unit tests                  | Database in integration tests |
| Time (Date.now, setTimeout)             | Pure functions                |
| File system, network                    | Value objects, entities       |
| Third-party libraries with side effects | Your own utility functions    |

### Mocking Patterns

```typescript
// Vitest/Jest mocking

// Mock a module
vi.mock('@/services/email', () => ({
  sendEmail: vi.fn().mockResolvedValue({ messageId: 'msg-123' }),
}))

// Mock with type safety
const mockSendEmail = vi
  .fn<[EmailParams], Promise<EmailResult>>()
  .mockResolvedValue({ messageId: 'msg-123' })

// Spy on a method (keeps original implementation by default)
const spy = vi.spyOn(orderService, 'calculateTotal')
expect(spy).toHaveBeenCalledWith(expect.objectContaining({ id: 'ord-1' }))

// Mock Date.now
vi.useFakeTimers()
vi.setSystemTime(new Date('2024-03-15T10:00:00Z'))
// ... run tests ...
vi.useRealTimers()

// Mock with different responses per call
mockFetch
  .mockResolvedValueOnce({ ok: true, json: () => ({ data: 'first' }) })
  .mockResolvedValueOnce({ ok: false, status: 500 })

// Reset between tests
beforeEach(() => {
  vi.clearAllMocks() // Clear call history and results
  // vi.resetAllMocks() // Also reset implementations
  // vi.restoreAllMocks() // Restore original implementations (for spies)
})
```

### Test Doubles Taxonomy (Gerard Meszaros)

| Double    | Purpose                               | Example                                  |
| --------- | ------------------------------------- | ---------------------------------------- |
| **Dummy** | Fills a parameter, never used         | `new OrderService(dummyLogger)`          |
| **Stub**  | Returns predetermined answers         | `findById: () => Promise.resolve(order)` |
| **Spy**   | Records calls for later assertion     | `vi.spyOn(service, 'save')`              |
| **Mock**  | Pre-programmed expectations           | `vi.fn().mockResolvedValue(result)`      |
| **Fake**  | Working but simplified implementation | In-memory repository instead of Prisma   |

```typescript
// Fake repository — useful for unit tests that need realistic behavior
class InMemoryOrderRepository implements OrderRepository {
  private orders = new Map<string, Order>()

  async findById(id: string): Promise<Order | null> {
    return this.orders.get(id) ?? null
  }

  async save(order: Order): Promise<void> {
    this.orders.set(order.id, order)
  }

  async delete(id: string): Promise<void> {
    this.orders.delete(id)
  }

  // Test helper
  seed(orders: Order[]): void {
    orders.forEach(o => this.orders.set(o.id, o))
  }
}
```

---

## 9. Testing Async Code

```typescript
// Awaiting promises
it('fetches and returns user', async () => {
  const user = await userService.findById('123')
  expect(user.name).toBe('John')
})

// Testing rejections
it('throws when user not found', async () => {
  await expect(userService.findById('nonexistent'))
    .rejects.toThrow(NotFoundError)
})

// waitFor — polling until condition is met (React Testing Library)
it('shows success message after async submission', async () => {
  await userEvent.click(screen.getByRole('button', { name: /submit/i }))

  await waitFor(() => {
    expect(screen.getByText(/success/i)).toBeInTheDocument()
  }, { timeout: 3000 })
})

// Fake timers — control time-dependent code
it('auto-saves after 5 seconds of inactivity', () => {
  vi.useFakeTimers()

  const save = vi.fn()
  render(<Editor onAutoSave={save} />)

  // Type something
  fireEvent.change(screen.getByRole('textbox'), { target: { value: 'hello' } })

  // Advance time
  vi.advanceTimersByTime(5000)

  expect(save).toHaveBeenCalledTimes(1)
  vi.useRealTimers()
})

// Testing event streams / Observables (Angular)
it('emits values over time', fakeAsync(() => {
  const values: number[] = []
  service.getStream().subscribe(v => values.push(v))

  tick(1000)
  expect(values).toEqual([1])

  tick(1000)
  expect(values).toEqual([1, 2])
}))
```

---

## 10. Coverage

### What Coverage Numbers Mean

| Metric        | What it measures            | Target |
| ------------- | --------------------------- | ------ |
| **Line**      | % of lines executed         | 80%+   |
| **Branch**    | % of if/else branches taken | 75%+   |
| **Function**  | % of functions called       | 85%+   |
| **Statement** | % of statements executed    | 80%+   |

### Coverage Is Not Quality

```typescript
// This test gets 100% coverage but tests NOTHING meaningful
it('runs the function', () => {
  const result = calculateOrderTotal(mockOrder)
  expect(result).toBeDefined() // So what? It could return garbage
})

// This test has real assertions on real behavior
it('calculates total including tax and discount', () => {
  const order = createOrder({
    items: [
      createOrderItem({ priceInCents: 1000, quantity: 2 }), // $20
      createOrderItem({ priceInCents: 500, quantity: 1 }), // $5
    ],
  })
  const result = calculateOrderTotal(order, { taxRate: 0.1, discount: 200 })
  expect(result).toBe(2530) // ($2500 - $200) * 1.1 = $2530
})
```

### What to Prioritize for Coverage

```
1. Business logic (domain services, calculations, rules)     — 90%+
2. API endpoints (request → response cycle)                   — 80%+
3. Error paths (validation, auth, not found)                  — 80%+
4. UI components (rendering, interactions)                    — 70%+
5. Utilities and helpers                                      — 90%+
6. Configuration and wiring (modules, DI)                     — don't test
```

---

## 11. Property-Based Testing

Instead of testing specific examples, test properties that should ALWAYS hold true:

```typescript
import { fc } from 'fast-check'

// Property: adding money and subtracting should return the original
it('add then subtract returns original amount', () => {
  fc.assert(
    fc.property(fc.integer({ min: 0, max: 1_000_000 }), cents => {
      const original = Money.fromCents(cents, 'USD')
      const added = original.add(Money.fromCents(500, 'USD'))
      const result = added.subtract(Money.fromCents(500, 'USD'))
      return result.equals(original)
    })
  )
})

// Property: sorting should be idempotent (sorting twice = sorting once)
it('sorting is idempotent', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), arr => {
      const once = [...arr].sort((a, b) => a - b)
      const twice = [...once].sort((a, b) => a - b)
      return JSON.stringify(once) === JSON.stringify(twice)
    })
  )
})

// Property: serialization roundtrip (serialize then deserialize = original)
it('Order survives JSON roundtrip', () => {
  fc.assert(
    fc.property(orderArbitrary(), order => {
      const serialized = OrderMapper.toDto(order)
      const deserialized = OrderMapper.toDomain(serialized)
      return order.equals(deserialized)
    })
  )
})
```

---

## 12. Common Mistakes

- **Testing implementation details** — don't assert on internal state, private methods, or call counts (unless that IS the behavior)
- **Copy-paste tests with minimal changes** — each test should test something meaningfully different
- **Ignoring async** — always `await` async operations, use `waitFor` in component tests
- **Snapshot testing as a crutch** — snapshots are fine for regression, but they don't verify behavior
- **Testing framework code** — don't test that React renders a div or that NestJS routes. Test YOUR code.
- **Global test state** — tests that depend on execution order are fragile. Each test should set up its own state.
- **Mocking everything** — if you mock the database, the repository, and the service, what are you testing? Just the wiring.
- **No error path tests** — only testing happy paths misses the most common source of bugs.

---

## 13. Process

1. Create the test file following the project's naming convention
2. Import only what's needed
3. Group with `describe`, name with behavior
4. Write tests: happy path → error paths → edge cases
5. Run tests to verify they pass
6. Report: how many tests, how many pass, coverage

---

## Output

```
## Test Summary

### Tests Written: X
- Unit: X tests for [what]
- Integration: X tests for [what]
- Component: X tests for [what]

### Coverage Impact
- Lines: X% → Y%
- Branches: X% → Y%

### Test Files
- `src/services/order.service.test.ts` — 8 tests (place, cancel, error paths)
- `src/components/checkout-form.test.tsx` — 5 tests (render, validation, submit)

### Verified
- ✓ All tests pass
- ✓ No flaky tests
- ✓ Matches project conventions
```

## Inspired By

- **TDD by Example** — Kent Beck (Red-Green-Refactor)
- **Working Effectively with Legacy Code** — Michael Feathers (Characterization Tests)
- **xUnit Test Patterns** — Gerard Meszaros (Test Doubles Taxonomy)
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas (Find Bugs Once)
- **Code Complete** — Steve McConnell
- **Unit Testing Principles, Practices, and Patterns** — Vladimir Khorikov
- **Growing Object-Oriented Software, Guided by Tests** — Steve Freeman & Nat Pryce
