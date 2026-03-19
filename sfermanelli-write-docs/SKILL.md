---
name: sfermanelli-write-docs
description: Generate clear, maintainable documentation for code — JSDoc/TSDoc, README, API docs, module overviews. Use this skill when the user asks to document code, add JSDoc, generate docs, write a README, explain an API surface, or says things like "document this", "add types docs", "this needs documentation", "write API docs". Distinct from explain-code (which is conversational) — this produces documentation artifacts that live in the codebase.
---

# Write Docs

Generate documentation that developers actually read. Good docs reduce onboarding time, prevent misuse, and serve as a contract between modules. Documentation should be accurate, concise, and maintained alongside the code it describes.

## Golden Rule

Documentation is a **code artifact** — it lives in the repo, gets reviewed in PRs, and must stay in sync with the code. Stale docs are worse than no docs.

> "A comment that is worth writing is worth writing well." — Robert C. Martin, Clean Code

---

## 1. What to Document (Ousterhout's Rules)

John Ousterhout identifies two types of knowledge in code:

- **Low-level** — What and How. This belongs in the code itself (names, types, structure).
- **High-level** — Why and What-for. This belongs in comments and documentation.

### Document These (High-Level Knowledge)

| What                                                                                  | Why it needs docs                                                                        | Example                                                                              |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Abstractions** — interface contracts, what a module promises                        | The type says `findById(id): Promise<T \| null>` but not WHY it returns null vs throwing | "Returns null if not found. Throws only on connection failure."                      |
| **Non-obvious behavior** — side effects, performance tradeoffs, ordering requirements | Code does what it does, but not why it does it that way                                  | "Uses eventual consistency — reads may not reflect the latest write for up to 500ms" |
| **Design decisions** — why this approach was chosen over alternatives                 | Code shows WHAT was built, not what was rejected                                         | "We use cursor pagination because offset pagination degrades on large datasets"      |
| **Invariants** — rules that must always hold true                                     | Constraints not enforceable by types alone                                               | "Amount must be in cents. Negative amounts represent refunds."                       |
| **Contracts** — what callers must do, what they can expect                            | Types catch structural issues but not semantic ones                                      | "Must be called after `initialize()`. Calling before init throws."                   |

### Don't Document These (Low-Level Knowledge)

```typescript
// Bad — repeats what the code already says
/** Gets the user by ID */
function getUserById(id: string): Promise<User | null>

/** Increments the counter */
counter++

/** The order items array */
items: OrderItem[]

// Good — adds knowledge the code doesn't convey
/**
 * Returns null if user was soft-deleted (not just missing).
 * Use `findActiveUserById` to exclude deleted users.
 */
function getUserById(id: string): Promise<User | null>
```

### The Precision Test (Ousterhout)

After writing a comment, ask: **"Could someone write the code just from this comment?"** If yes, the comment captures the full abstraction. If not, it's missing something.

```typescript
// Fails precision test — doesn't tell you what "ready" means
/** Returns true when the order is ready. */
function isReady(order: Order): boolean

// Passes precision test — you could implement this from the comment
/**
 * Returns true when the order has been paid AND all items are in stock.
 * Does NOT check shipping availability — that's a separate step.
 */
function isReady(order: Order): boolean
```

---

## 2. TSDoc / JSDoc

The primary documentation tool for TypeScript codebases. Document the **public API surface** — don't document private internals that change frequently.

### When to Document

- Exported functions, classes, interfaces, and types
- Non-obvious parameters, return values, and side effects
- Complex generics or type constraints
- Configuration objects and their defaults
- Domain-specific terminology

### When NOT to Document

- Self-explanatory code (`getName(): string` doesn't need a doc)
- Private methods with clear names
- Re-exports or simple wrappers
- Obvious getters/setters

### Function Documentation

````typescript
/**
 * Calculates the total price for an order including tax and discounts.
 *
 * Discounts are applied BEFORE tax (pre-tax discount model).
 * All monetary calculations use integer cents to avoid floating-point errors.
 *
 * @param items - Line items in the order. Must not be empty.
 * @param options - Pricing configuration
 * @param options.taxRate - Tax rate as decimal (e.g., 0.21 for 21%). Must be >= 0.
 * @param options.discount - Optional discount to apply before tax
 * @returns The final price in cents — always rounded up to avoid underbilling
 *
 * @throws {InvalidOrderError} When items array is empty
 * @throws {PricingError} When calculated total is negative (over-discounted)
 *
 * @example
 * ```typescript
 * const total = calculateOrderTotal(
 *   [{ productId: 'abc', price: 1999, quantity: 2 }],
 *   { taxRate: 0.21 }
 * )
 * // total = 4838 (1999 * 2 * 1.21 = 4837.58, rounded up to 4838)
 * ```
 *
 * @see {@link applyDiscount} for discount calculation logic
 * @see {@link Money} for the monetary value type used internally
 */
function calculateOrderTotal(items: OrderItem[], options: PricingOptions): number
````

### TSDoc Tags Reference

| Tag                | Use for                                                           | Example                                                         |
| ------------------ | ----------------------------------------------------------------- | --------------------------------------------------------------- |
| `@param`           | Function parameters — include nested properties with dot notation | `@param options.taxRate - Tax rate as decimal`                  |
| `@returns`         | Return value — explain what it represents, not just the type      | `@returns Price in cents, rounded up`                           |
| `@throws`          | Exceptions — include the error class and when it fires            | `@throws {NotFoundError} When order doesn't exist`              |
| `@example`         | Usage example — keep it minimal, runnable, with expected output   | See above                                                       |
| `@remarks`         | Extended explanation — design decisions, caveats, performance     | Algorithm details, tradeoffs                                    |
| `@see`             | Cross-references to related functions or external docs            | `@see {@link Money}`                                            |
| `@deprecated`      | Mark deprecated — ALWAYS include the replacement                  | `@deprecated Use {@link calculateV2} instead. Removes in v4.0.` |
| `@typeParam`       | Generic type parameters                                           | `@typeParam T - The entity type`                                |
| `@defaultValue`    | Default values for optional parameters                            | `@defaultValue 20`                                              |
| `@internal`        | Not part of public API — may change without notice                | Internal helpers                                                |
| `@alpha` / `@beta` | API stability markers                                             | Experimental features                                           |

### Class Documentation

````typescript
/**
 * Manages WebSocket connections with automatic reconnection and heartbeat.
 *
 * @remarks
 * Uses exponential backoff for reconnection attempts (1s, 2s, 4s, 8s, 16s).
 * The connection is considered dead after {@link ConnectionManager.MAX_RETRIES}
 * failed attempts, at which point it emits a `disconnected` event.
 *
 * This class follows the Observer pattern — subscribe to events for state changes
 * rather than polling the connection status.
 *
 * @example
 * ```typescript
 * const manager = new ConnectionManager({
 *   url: 'wss://api.example.com',
 *   heartbeatIntervalMs: 30_000
 * })
 *
 * manager.on('message', (data) => console.log(data))
 * manager.on('disconnected', () => showReconnectBanner())
 *
 * await manager.connect()
 * ```
 */
class ConnectionManager {
  /** Maximum reconnection attempts before giving up. */
  static readonly MAX_RETRIES = 5

  /**
   * Creates a new connection manager.
   *
   * @param config - Connection configuration
   * @param config.url - WebSocket URL (wss:// for production)
   * @param config.heartbeatIntervalMs - Interval between heartbeat pings.
   *   @defaultValue 30000
   */
  constructor(private config: ConnectionConfig) {}

  /**
   * Establishes the WebSocket connection.
   *
   * @remarks
   * Resolves when the connection is established and the first heartbeat succeeds.
   * Rejects if the initial connection fails (does NOT auto-retry the initial connection).
   *
   * @throws {ConnectionError} When the WebSocket handshake fails
   * @throws {AuthenticationError} When the server rejects the auth token
   */
  async connect(): Promise<void> { ... }
}
````

### Interface and Type Documentation

````typescript
/**
 * Configuration for the payment processing pipeline.
 *
 * @remarks
 * All monetary values are in the smallest currency unit (cents for USD, pence for GBP).
 * The processor validates amounts against {@link PaymentLimits} before execution.
 *
 * **Thread safety:** This configuration is read-only after initialization.
 * Modifying it at runtime has undefined behavior.
 */
interface PaymentConfig {
  /** Payment gateway to use. Defaults to the primary gateway for the store's region. */
  gateway: PaymentGateway

  /**
   * Maximum amount per transaction in cents.
   * Transactions exceeding this amount are automatically split.
   * @defaultValue 100_000_00 ($100,000)
   */
  maxAmount?: number

  /**
   * Retry configuration for failed charges.
   * Set to `false` to disable retries entirely.
   *
   * @remarks
   * Only retries on transient errors (network timeout, gateway 503).
   * Never retries on card-declined or insufficient-funds.
   */
  retry?: RetryConfig | false

  /**
   * Webhook URL for payment status notifications.
   * Must be HTTPS in production. HTTP allowed only in development.
   */
  webhookUrl?: string
}

/**
 * Represents a monetary value with currency.
 *
 * @remarks
 * Immutable value object. All arithmetic operations return new instances.
 * Uses integer arithmetic internally to avoid floating-point errors.
 *
 * @example
 * ```typescript
 * const price = Money.fromUnits(49.99, 'USD')
 * const withTax = price.multiply(1.21)
 * console.log(withTax.format()) // "$60.49"
 * ```
 */
````

### Enum / Union Documentation

````typescript
/**
 * Lifecycle states of an order.
 *
 * State transitions:
 * ```
 * DRAFT → PLACED → PROCESSING → SHIPPED → DELIVERED
 *                                    ↓
 *   DRAFT → CANCELLED          RETURNED
 *   PLACED → CANCELLED
 * ```
 *
 * @remarks
 * Only `DRAFT` and `PLACED` orders can be cancelled.
 * Cancellation after `PROCESSING` requires a refund workflow.
 */
enum OrderStatus {
  /** Initial state. Order is being built, not yet submitted. */
  Draft = 'DRAFT',

  /** Order submitted and payment authorized. Awaiting processing. */
  Placed = 'PLACED',

  /** Order is being prepared for shipment. Cannot be cancelled. */
  Processing = 'PROCESSING',

  /** Order has been handed off to the shipping carrier. */
  Shipped = 'SHIPPED',

  /** Order delivered to the customer. Terminal state. */
  Delivered = 'DELIVERED',

  /** Order cancelled before processing. Terminal state. */
  Cancelled = 'CANCELLED',

  /** Order returned after delivery. Triggers refund workflow. */
  Returned = 'RETURNED',
}
````

### Generic Documentation

````typescript
/**
 * A paginated response wrapper for API list endpoints.
 *
 * @typeParam T - The entity type contained in the response
 *
 * @example
 * ```typescript
 * // In a controller
 * const response: PaginatedResponse<OrderDto> = {
 *   data: orders.map(OrderMapper.toDto),
 *   pagination: { page: 1, pageSize: 20, total: 156, totalPages: 8 }
 * }
 * ```
 */
interface PaginatedResponse<T> {
  /** The page of results. Empty array if no results. */
  data: T[]
  pagination: PaginationMeta
}

/**
 * Extracts the resolved type from a Promise, or returns the type as-is.
 *
 * @typeParam T - The type to unwrap
 *
 * @example
 * ```typescript
 * type A = Awaited<Promise<string>>          // string
 * type B = Awaited<Promise<Promise<number>>> // number (recursive)
 * type C = Awaited<string>                   // string (no-op)
 * ```
 */
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T
````

---

## 3. README / Module Documentation

### Module README Structure

For internal modules or packages within a monorepo:

```markdown
# Module Name

One sentence: what this module does and why it exists.

## Usage

\`\`\`typescript
import { createOrder } from '@myapp/orders'

const order = await createOrder({
customerId: 'cust_123',
items: [{ productId: 'prod_456', quantity: 2 }]
})
\`\`\`

## API

Brief description of the public API surface.

### `createOrder(input: CreateOrderInput): Promise<Order>`

Creates a new order. Validates stock availability before creating.

### `cancelOrder(orderId: string, reason: string): Promise<void>`

Cancels an order. Only works for orders in DRAFT or PLACED status.

## Architecture Decisions

- **Why event-driven** — Side effects (emails, analytics) are decoupled via domain events
  so the core order flow stays fast and testable.
- **Why cursor pagination** — Offset pagination degrades on large datasets and gives
  inconsistent results when records are added during pagination.

## Known Limitations

- Maximum 50 items per order (validated at creation)
- No partial cancellation — cancel the whole order and recreate
- Currency must be the same for all items (no multi-currency orders)

## Dependencies

- `@myapp/inventory` — Stock availability checks
- `@myapp/payments` — Payment authorization
- `stripe` — Payment gateway (via adapter)
```

### Project README Structure

```markdown
# Project Name

One-line description of what this project does.

## Quick Start

\`\`\`bash
git clone <repo>
cp .env.example .env
npm install
npx prisma migrate dev
npm run dev
\`\`\`

## Architecture

Brief overview — what are the main pieces and how do they connect.

\`\`\`
┌─────────┐ ┌──────────┐ ┌──────────┐
│ Client │────▶│ API │────▶│ Database │
│ (Next.js)│ │ (NestJS) │ │ (Postgres)│
└─────────┘ └────┬─────┘ └──────────┘
│
▼
┌──────────┐
│ Queue │
│ (Redis) │
└──────────┘
\`\`\`

## Development

### Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+

### Setup

Detailed setup instructions.

### Running Tests

\`\`\`bash
npm run test # Unit tests
npm run test:e2e # End-to-end tests
npm run test:cov # Coverage report
\`\`\`

### Common Tasks

- **Generate migration:** `npx prisma migrate dev --name <name>`
- **Seed database:** `npm run db:seed`
- **Generate types:** `npm run codegen`

## Deployment

How to deploy — or link to deployment docs.

## Environment Variables

| Variable              | Required | Description                  | Example                                      |
| --------------------- | -------- | ---------------------------- | -------------------------------------------- |
| `DATABASE_URL`        | Yes      | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/mydb` |
| `REDIS_URL`           | Yes      | Redis connection string      | `redis://localhost:6379`                     |
| `STRIPE_SECRET_KEY`   | Yes      | Stripe API key               | `sk_test_...`                                |
| `NEXT_PUBLIC_API_URL` | Yes      | API base URL                 | `http://localhost:3001`                      |
```

---

## 4. API Documentation

### REST Endpoint Documentation

````typescript
/**
 * Creates a new store for the authenticated user.
 *
 * @route POST /api/stores
 * @auth Required — Bearer token (must have 'stores:create' permission)
 *
 * @param body.name - Store display name (3-50 characters, trimmed)
 * @param body.slug - URL-friendly identifier. Auto-generated from name if omitted.
 *   Must be unique, lowercase, 3-30 chars, only letters/numbers/hyphens.
 * @param body.plan - Subscription plan. @defaultValue "free"
 * @param body.currency - Store currency (ISO 4217). @defaultValue "USD"
 *
 * @returns 201 — The created store with generated ID. Location header points to new resource.
 * @returns 400 — Validation error (name too short, invalid slug format)
 * @returns 401 — Missing or invalid auth token
 * @returns 409 — Store with this slug already exists
 * @returns 429 — Rate limited (max 5 stores per hour per user)
 *
 * @example
 * ```bash
 * curl -X POST /api/stores \
 *   -H "Authorization: Bearer $TOKEN" \
 *   -H "Content-Type: application/json" \
 *   -d '{"name": "My Store", "plan": "pro"}'
 *
 * # Response (201 Created)
 * # Location: /api/stores/store_abc123
 * {
 *   "data": {
 *     "id": "store_abc123",
 *     "name": "My Store",
 *     "slug": "my-store",
 *     "plan": "pro",
 *     "createdAt": "2024-03-15T10:30:00Z"
 *   }
 * }
 * ```
 */
````

### OpenAPI / Swagger Integration

```typescript
// NestJS + Swagger decorators — generate OpenAPI spec from code
@ApiTags('orders')
@Controller('orders')
export class OrderController {
  @Post()
  @ApiOperation({ summary: 'Create a new order' })
  @ApiBody({ type: CreateOrderDto })
  @ApiResponse({ status: 201, description: 'Order created', type: OrderResponseDto })
  @ApiResponse({ status: 400, description: 'Validation error' })
  @ApiResponse({ status: 409, description: 'Insufficient stock' })
  @ApiBearerAuth()
  async create(@Body() dto: CreateOrderDto): Promise<OrderResponseDto> { ... }
}

// DTO with validation + documentation in one place
class CreateOrderDto {
  @ApiProperty({
    description: 'Products to order',
    type: [OrderItemDto],
    minItems: 1,
    maxItems: 50
  })
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(50)
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[]

  @ApiProperty({
    description: 'Discount code to apply',
    required: false,
    example: 'SUMMER20'
  })
  @IsOptional()
  @IsString()
  @MaxLength(20)
  discountCode?: string
}
```

---

## 5. Changelog and ADR Documentation

### CHANGELOG.md

```markdown
# Changelog

All notable changes to this project will be documented in this file.

Format based on [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

### Added

- Cursor-based pagination for order listing endpoint

### Changed

- Order total calculation now uses integer cents instead of floats

### Fixed

- Race condition when two users cancel the same order simultaneously

## [2.1.0] - 2024-03-15

### Added

- Bulk order creation endpoint (POST /api/orders/batch)
- Webhook notifications for order status changes

### Deprecated

- `GET /api/orders?offset=N` — use cursor pagination instead. Will be removed in v3.0.

### Security

- Fixed IDOR vulnerability in order detail endpoint (CVE-2024-XXXX)
```

### Architecture Decision Record (ADR)

```markdown
# ADR-003: Use Cursor Pagination for Order Listing

## Status

Accepted (2024-03-10)

## Context

The orders table has grown to 2M+ rows. Offset pagination (`LIMIT/OFFSET`)
becomes progressively slower as the offset increases because the database
must scan and discard all preceding rows.

Users report 3-5 second load times on page 50+ of order lists.

## Decision

Switch the `/api/orders` endpoint from offset to cursor-based pagination.
The cursor is an opaque base64-encoded token containing the sort key values
of the last returned record.

## Consequences

### Positive

- Consistent O(1) query time regardless of page depth
- No duplicate/missing records when data changes between pages
- Works naturally with infinite scroll UIs

### Negative

- Cannot jump to arbitrary page (no "go to page 50")
- Consumers must update to use cursor instead of page number
- More complex implementation in the repository layer

### Migration

- v2.1: Add cursor support alongside offset (both work)
- v2.2: Deprecate offset with Sunset header
- v3.0: Remove offset support

## Alternatives Considered

### Keyset pagination with visible keys

Expose the sort key directly (e.g., `?after_id=123`). Rejected because
it leaks internal IDs and breaks if the sort key changes.

### Keep offset with caching

Cache common page offsets. Rejected because it doesn't solve the consistency
issue and adds cache invalidation complexity.
```

---

## 6. Documentation Levels (The Four Types)

Inspired by Divio's documentation system:

| Type             | Purpose                                          | Audience             | Example                             |
| ---------------- | ------------------------------------------------ | -------------------- | ----------------------------------- |
| **Tutorial**     | Learning-oriented. Walk through a complete task. | Newcomers            | "Build your first order endpoint"   |
| **How-to Guide** | Task-oriented. Solve a specific problem.         | Practitioners        | "How to add a new payment provider" |
| **Reference**    | Information-oriented. Describe the machinery.    | Experienced users    | TSDoc, API spec, config reference   |
| **Explanation**  | Understanding-oriented. Clarify concepts.        | Anyone seeking depth | ADRs, architecture overviews        |

### When to write each

- **New feature** → Tutorial (if complex) + Reference (TSDoc) + How-to (if config needed)
- **Bugfix** → No docs unless it changes behavior
- **Architecture change** → ADR + update Explanation docs
- **New config option** → Reference only

---

## 7. Documentation Anti-Patterns

### Don't Document the Obvious

```typescript
// Bad — adds noise, zero value
/** The user's name. */
name: string

/** Gets the user's email. */
getEmail(): string

// Good — documents what isn't obvious
/** Full legal name as it appears on billing documents. Not the display name. */
legalName: string

/** Primary verified email. Null if the user signed up via OAuth without email scope. */
getVerifiedEmail(): string | null
```

### Don't Let Docs Lie

```typescript
// Bad — doc says one thing, code does another
/** Returns the user's age. */
function getUser(): User { ... }

// This is worse than no docs — it actively misleads
```

### Don't Write Novels

```typescript
// Bad — restating the signature in prose
/**
 * This function is used to validate the email address of a user.
 * It takes a string parameter which should be the email address
 * that needs to be validated. The function uses a regular expression
 * to check if the email address is in a valid format. If the email
 * is valid, it returns true. If not, it returns false.
 */
function isValidEmail(email: string): boolean

// Good — adds what the types don't tell you
/**
 * Validates email format per RFC 5322.
 * Does NOT check deliverability or MX records.
 * Does NOT normalize (use {@link normalizeEmail} first if comparing).
 */
function isValidEmail(email: string): boolean
```

### Don't Use Comments as Version Control

```typescript
// Bad — that's what git is for
// Added by John on 2024-03-15
// Modified by Jane on 2024-03-20
// Previous implementation used regex

// Bad — commented-out code
// function oldImplementation() { ... }
```

### Don't Document Around Bad Code

```typescript
// Bad — the comment is a code smell
// We need to check if the order is valid, which means checking
// that it has items, that the total is positive, that the customer
// exists, and that the payment method is valid
if (order.items.length > 0 && order.total > 0 && customer && paymentMethod.isValid) { ... }

// Good — make the code self-documenting
if (order.isValidForCheckout()) { ... }
```

---

## 8. Process

1. **Read the code** — Understand what it does before documenting it
2. **Identify the audience** — Is this for consumers of the API or maintainers of the internals?
3. **Choose the documentation type** — Reference, tutorial, how-to, or explanation?
4. **Document the public surface** — Exports, interfaces, configuration
5. **Add examples** — One minimal, runnable example per major function. Include expected output.
6. **Document the non-obvious** — Side effects, invariants, design decisions, tradeoffs
7. **Skip the obvious** — Only document what the types and names don't already tell you
8. **Verify accuracy** — Run the examples, check the types match the docs, verify @see links

---

## Output

```
## Documentation Summary

### TSDoc Added (X items)
- `calculateOrderTotal` — params, returns, throws, example
- `PaymentConfig` interface — all properties with invariants
- `ConnectionManager` class — constructor, events, lifecycle, example
- `OrderStatus` enum — state transition diagram, rules

### Module Docs
- Added README.md for `packages/payments`
- Updated architecture diagram in root README

### ADRs
- ADR-003: Cursor pagination (context, decision, consequences)

### Improvements
- Removed 5 stale @param tags that didn't match current signatures
- Added @deprecated to 3 functions with migration paths and sunset dates
- Fixed 2 @example blocks that didn't compile

### Verified
- All @example blocks compile and produce expected output
- No broken @see references
- No stale documentation (compared against current signatures)
```

## Inspired By

- **A Philosophy of Software Design** — John Ousterhout (Chapters 12-13: Why Write Comments, Comments Should Describe Things That Aren't Obvious)
- **Clean Code** — Robert C. Martin (Chapter 4: Comments)
- **TSDoc Specification** — microsoft.github.io/tsdoc
- **Docs for Developers** — Jared Bhatti, Zachary Sarah Corleissen, Jen Lambourne, David Nunez, Heidi Waterhouse
- **The Documentation System** — Divio (tutorials, how-to, reference, explanation)
- **Living Documentation** — Cyrille Martraire
- **The Art of Readable Code** — Dustin Boswell & Trevor Foucher
