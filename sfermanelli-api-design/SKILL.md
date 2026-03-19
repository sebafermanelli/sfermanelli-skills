---
name: sfermanelli-api-design
description: Design clean, consistent, and secure APIs — REST endpoints, GraphQL schemas, internal service contracts, and DTOs. Use this skill when the user asks to design an API, create endpoints, define a schema, plan an API surface, or says things like "design the API", "what should the endpoint look like", "create the routes", "define the contract", "design the schema". Distinct from write-docs (which documents existing code) — this designs new API contracts from requirements.
---

# API Design

Design APIs that are intuitive, consistent, and hard to misuse. A good API makes the right thing easy and the wrong thing impossible. Consumers should be able to guess the endpoint, the shape, and the behavior before reading the docs.

## Golden Rule

**Design for the consumer, not the implementation.** The API shape should reflect what the consumer needs to do, not how the server is structured internally. Never expose implementation details through the API surface.

---

## 1. REST API Design

### Resource Naming

```
# Resources are nouns, not verbs
GET    /api/orders              # List orders
POST   /api/orders              # Create order
GET    /api/orders/:id          # Get order
PATCH  /api/orders/:id          # Update order
DELETE /api/orders/:id          # Delete order

# Nested resources for clear ownership
GET    /api/stores/:storeId/orders       # Orders for a store
POST   /api/stores/:storeId/orders       # Create order in store

# Actions that don't map to CRUD — use verbs as sub-resources
POST   /api/orders/:id/cancel            # Cancel order
POST   /api/orders/:id/refund            # Refund order
POST   /api/auth/login                   # Login (action, not resource)
```

### Naming Rules

| Do                                    | Don't                                      |
| ------------------------------------- | ------------------------------------------ |
| Plural nouns: `/orders`               | Singular: `/order`                         |
| Kebab-case: `/order-items`            | camelCase: `/orderItems`                   |
| Flat when possible: `/orders?store=X` | Deep nesting: `/stores/X/orders/Y/items/Z` |
| Consistent across all endpoints       | Mixed conventions                          |

### HTTP Methods

| Method | Semantics       | Idempotent | Safe | Request Body |
| ------ | --------------- | ---------- | ---- | ------------ |
| GET    | Read            | Yes        | Yes  | No           |
| POST   | Create / Action | No         | No   | Yes          |
| PUT    | Full replace    | Yes        | No   | Yes          |
| PATCH  | Partial update  | Yes        | No   | Yes          |
| DELETE | Remove          | Yes        | No   | Optional     |

### Status Codes

```typescript
// Success
200 // OK — GET, PATCH, DELETE with body
201 // Created — POST that creates a resource (include Location header)
204 // No Content — DELETE, PUT with no body

// Redirection
301 // Moved Permanently — resource URL changed
304 // Not Modified — cache is still valid (ETag/If-None-Match)

// Client Errors
400 // Bad Request — malformed JSON, missing required field
401 // Unauthorized — missing or invalid authentication
403 // Forbidden — authenticated but not authorized for this resource
404 // Not Found — resource doesn't exist (also use to hide existence from unauthorized users)
405 // Method Not Allowed — e.g., DELETE on a read-only resource
409 // Conflict — duplicate entry, state conflict, optimistic locking failure
413 // Payload Too Large — request body exceeds limit
422 // Unprocessable Entity — valid JSON but invalid business logic
429 // Too Many Requests — rate limit exceeded (include Retry-After header)

// Server Errors
500 // Internal Server Error — unexpected failure, never expose internals
502 // Bad Gateway — upstream service failure
503 // Service Unavailable — overloaded, maintenance (include Retry-After header)
504 // Gateway Timeout — upstream service timeout
```

### Response Envelope

```typescript
// Single resource
interface ApiResponse<T> {
  data: T
  meta?: Record<string, unknown> // Extra info (deprecation notices, rate limit, etc.)
}

// Collection with pagination
interface PaginatedResponse<T> {
  data: T[]
  pagination: {
    total: number
    page: number
    pageSize: number
    totalPages: number
    hasNext: boolean
    hasPrevious: boolean
  }
}

// Cursor-based pagination (for real-time/infinite scroll)
interface CursorPaginatedResponse<T> {
  data: T[]
  pagination: {
    cursor: string | null // null = no more results
    hasMore: boolean
    limit: number
  }
}

// Error — consistent across ALL endpoints
interface ApiErrorResponse {
  error: {
    code: string // Machine-readable: "VALIDATION_ERROR"
    message: string // Human-readable: "Email is required"
    details?: ErrorDetail[] // Field-level errors (for validation)
    requestId?: string // For support/debugging
  }
}

interface ErrorDetail {
  field: string // JSON path: "items[0].quantity"
  message: string // "Must be a positive number"
  code: string // "POSITIVE_NUMBER"
  received?: unknown // What the server actually received
}
```

### Error Codes Catalog

Define a finite set of error codes. Consumers can programmatically react to these.

```typescript
// Define ALL possible error codes as a union type
type ErrorCode =
  // Validation
  | 'VALIDATION_ERROR'
  | 'INVALID_FORMAT'
  | 'REQUIRED_FIELD'
  | 'OUT_OF_RANGE'
  // Auth
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'
  | 'TOKEN_EXPIRED'
  | 'INVALID_CREDENTIALS'
  // Resources
  | 'NOT_FOUND'
  | 'ALREADY_EXISTS'
  | 'CONFLICT'
  | 'GONE'
  // Business logic
  | 'INSUFFICIENT_STOCK'
  | 'ORDER_NOT_MODIFIABLE'
  | 'PAYMENT_FAILED'
  | 'LIMIT_EXCEEDED'
  // Server
  | 'INTERNAL_ERROR'
  | 'SERVICE_UNAVAILABLE'
  | 'UPSTREAM_ERROR'

// Consistent error factory
function createApiError(
  statusCode: number,
  code: ErrorCode,
  message: string,
  details?: ErrorDetail[]
): ApiErrorResponse {
  return {
    error: {
      code,
      message,
      details,
      requestId: getRequestId(),
    },
  }
}
```

---

## 2. Validation Layer

### Zod Schema Design

```typescript
import { z } from 'zod'

// Reusable primitives — define once, use everywhere
const schemas = {
  id: z.string().uuid(),
  email: z
    .string()
    .email()
    .max(255)
    .transform(v => v.toLowerCase().trim()),
  money: z.object({
    amount: z.number().int().nonnegative(),
    currency: z.string().length(3).toUpperCase(),
  }),
  pagination: z.object({
    page: z.coerce.number().int().positive().default(1),
    pageSize: z.coerce.number().int().min(1).max(100).default(20),
  }),
  sort: z
    .string()
    .regex(/^\w+:(asc|desc)$/)
    .optional(),
  dateRange: z
    .object({
      from: z.coerce.date(),
      to: z.coerce.date(),
    })
    .refine(d => d.to > d.from, { message: 'End date must be after start date' }),
}

// Compose schemas from primitives
const CreateOrderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: schemas.id,
        quantity: z.number().int().positive().max(100),
      })
    )
    .min(1, 'Order must have at least one item')
    .max(50, 'Maximum 50 items per order'),
  shippingAddress: AddressSchema,
  discountCode: z.string().min(3).max(20).optional(),
  notes: z.string().max(500).optional(),
})

const UpdateOrderSchema = CreateOrderSchema.partial().refine(data => Object.keys(data).length > 0, {
  message: 'At least one field must be provided',
})

const ListOrdersSchema = z.object({
  ...schemas.pagination.shape,
  status: z.enum(['draft', 'placed', 'shipped', 'delivered', 'cancelled']).optional(),
  customerId: schemas.id.optional(),
  createdAfter: z.coerce.date().optional(),
  createdBefore: z.coerce.date().optional(),
  sort: schemas.sort.default('createdAt:desc'),
  search: z.string().max(100).optional(),
})

// Infer TypeScript types from schemas — single source of truth
type CreateOrderRequest = z.infer<typeof CreateOrderSchema>
type UpdateOrderRequest = z.infer<typeof UpdateOrderSchema>
type ListOrdersParams = z.infer<typeof ListOrdersSchema>
```

### Validation Middleware

```typescript
// Generic validation middleware — works with any Zod schema
function validate<T extends z.ZodType>(schema: T, source: 'body' | 'query' | 'params' = 'body') {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req[source])

    if (!result.success) {
      return res.status(400).json(formatZodError(result.error))
    }

    req.validated = result.data
    next()
  }
}

function formatZodError(error: z.ZodError): ApiErrorResponse {
  return {
    error: {
      code: 'VALIDATION_ERROR',
      message: 'Invalid request data',
      details: error.issues.map(issue => ({
        field: issue.path.join('.'),
        message: issue.message,
        code: issue.code.toUpperCase(),
      })),
    },
  }
}

// Usage
router.post('/orders', validate(CreateOrderSchema), handleCreateOrder)
router.get('/orders', validate(ListOrdersSchema, 'query'), handleListOrders)
```

### Discriminated Unions for Polymorphic Requests

```typescript
// When the request shape depends on a type field
const PaymentMethodSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('credit_card'),
    cardNumber: z.string().regex(/^\d{16}$/),
    expiryMonth: z.number().int().min(1).max(12),
    expiryYear: z.number().int().min(2024),
    cvv: z.string().regex(/^\d{3,4}$/),
  }),
  z.object({
    type: z.literal('bank_transfer'),
    bankCode: z.string(),
    accountNumber: z.string(),
  }),
  z.object({
    type: z.literal('wallet'),
    walletId: z.string().uuid(),
    pin: z.string().length(6),
  }),
])

type PaymentMethod = z.infer<typeof PaymentMethodSchema>
// TypeScript knows: if type === 'credit_card', then cardNumber exists
```

---

## 3. DTOs and Mapping

### DTO Design Principles

```typescript
// DTOs are pure data — no methods, no behavior, fully serializable
// Use them at the boundary between your domain and the outside world

// Input DTO (what the client sends)
interface CreateProductInput {
  name: string
  description: string
  priceInCents: number
  currency: string
  categoryId: string
  tags?: string[]
}

// Output DTO (what the client receives)
interface ProductDto {
  id: string
  name: string
  description: string
  price: {
    amount: number // In cents
    currency: string
    formatted: string // "$49.99" — convenience for the consumer
  }
  category: {
    id: string
    name: string
  }
  tags: string[]
  createdAt: string // ISO 8601 always
  updatedAt: string
}

// Summary DTO (for lists — fewer fields, less data)
interface ProductSummaryDto {
  id: string
  name: string
  price: {
    amount: number
    currency: string
    formatted: string
  }
  thumbnailUrl: string | null
}
```

### Mapper Pattern

```typescript
// Mapper class — translates between domain and DTOs
// Keep mappers as pure functions, no side effects
class ProductMapper {
  static toDto(product: Product): ProductDto {
    return {
      id: product.id.value,
      name: product.name,
      description: product.description,
      price: {
        amount: product.price.cents,
        currency: product.price.currency,
        formatted: product.price.format(),
      },
      category: {
        id: product.category.id.value,
        name: product.category.name,
      },
      tags: product.tags.map(t => t.value),
      createdAt: product.createdAt.toISOString(),
      updatedAt: product.updatedAt.toISOString(),
    }
  }

  static toSummaryDto(product: Product): ProductSummaryDto {
    return {
      id: product.id.value,
      name: product.name,
      price: {
        amount: product.price.cents,
        currency: product.price.currency,
        formatted: product.price.format(),
      },
      thumbnailUrl: product.thumbnail?.url ?? null,
    }
  }

  static toDomain(input: CreateProductInput): Product {
    return Product.create({
      name: input.name,
      description: input.description,
      price: Money.fromCents(input.priceInCents, input.currency),
      categoryId: new CategoryId(input.categoryId),
      tags: input.tags?.map(t => new Tag(t)) ?? [],
    })
  }
}
```

### Response Builder

```typescript
// Consistent response construction
class ResponseBuilder {
  static ok<T>(data: T, meta?: Record<string, unknown>): ApiResponse<T> {
    return { data, ...(meta && { meta }) }
  }

  static created<T>(data: T): ApiResponse<T> {
    return { data }
  }

  static paginated<T>(
    data: T[],
    total: number,
    page: number,
    pageSize: number
  ): PaginatedResponse<T> {
    const totalPages = Math.ceil(total / pageSize)
    return {
      data,
      pagination: {
        total,
        page,
        pageSize,
        totalPages,
        hasNext: page < totalPages,
        hasPrevious: page > 1,
      },
    }
  }

  static cursor<T>(data: T[], cursor: string | null, limit: number): CursorPaginatedResponse<T> {
    return {
      data,
      pagination: {
        cursor,
        hasMore: cursor !== null,
        limit,
      },
    }
  }
}

// Usage in handlers
async function handleGetOrder(req: Request, res: Response) {
  const order = await orderService.findById(req.params.id)
  if (!order) return res.status(404).json(createApiError(404, 'NOT_FOUND', 'Order not found'))
  return res.json(ResponseBuilder.ok(OrderMapper.toDto(order)))
}

async function handleListOrders(req: Request, res: Response) {
  const params = req.validated as ListOrdersParams
  const { orders, total } = await orderService.list(params)
  return res.json(
    ResponseBuilder.paginated(
      orders.map(OrderMapper.toSummaryDto),
      total,
      params.page,
      params.pageSize
    )
  )
}
```

---

## 4. Advanced Patterns

### Filtering, Sorting, Pagination

```
# Filtering — use query params with clear names
GET /api/orders?status=pending&createdAfter=2024-01-01&minTotal=5000

# Multiple values — comma-separated or repeated params
GET /api/orders?status=pending,processing
GET /api/orders?status=pending&status=processing

# Sorting — explicit syntax, support multiple fields
GET /api/orders?sort=createdAt:desc
GET /api/orders?sort=status:asc,createdAt:desc

# Cursor-based pagination (for real-time data, infinite scroll)
GET /api/orders?cursor=eyJpZCI6MTIzfQ&limit=20
# The cursor is an opaque token — the client never parses it

# Offset-based pagination (for static data, admin panels)
GET /api/orders?page=2&pageSize=20
```

### Search

```typescript
// Full-text search endpoint
// GET /api/products/search?q=wireless+headphones&category=electronics&sort=relevance:desc

const SearchSchema = z.object({
  q: z.string().min(2).max(100),
  category: z.string().optional(),
  minPrice: z.coerce.number().nonnegative().optional(),
  maxPrice: z.coerce.number().nonnegative().optional(),
  sort: z
    .enum(['relevance:desc', 'price:asc', 'price:desc', 'newest:desc'])
    .default('relevance:desc'),
  page: z.coerce.number().int().positive().default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
})

// Search response includes relevance info
interface SearchResponse<T> extends PaginatedResponse<T> {
  meta: {
    query: string
    totalResults: number
    facets?: Record<string, Array<{ value: string; count: number }>>
  }
}
```

### Bulk Operations

```typescript
// POST /api/orders/batch — for multiple operations in one request
interface BatchRequest<T> {
  operations: Array<{
    method: 'create' | 'update' | 'delete'
    id?: string // Required for update/delete
    data?: Partial<T> // Required for create/update
  }>
}

interface BatchResponse<T> {
  results: Array<{
    index: number
    status: 'success' | 'error'
    data?: T
    error?: { code: string; message: string }
  }>
  summary: {
    total: number
    succeeded: number
    failed: number
  }
}

// Rules:
// 1. Each operation is independent — one failure doesn't roll back others
// 2. Max operations per batch (e.g., 100) to prevent abuse
// 3. Return individual status for each operation
// 4. Return HTTP 200 even if some operations fail (the batch itself succeeded)
// 5. Return HTTP 400 only if the batch request itself is malformed
```

### Idempotency

```typescript
// For non-idempotent operations (POST), use idempotency keys
// The client generates a unique key and sends it in the header
// If the server sees the same key again, it returns the cached response

// Header: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

interface IdempotencyRecord {
  key: string
  statusCode: number
  body: unknown
  createdAt: Date
  expiresAt: Date
}

async function withIdempotency(
  key: string,
  handler: () => Promise<{ statusCode: number; body: unknown }>
): Promise<{ statusCode: number; body: unknown }> {
  // Check if we already processed this key
  const existing = await idempotencyStore.get(key)
  if (existing) return { statusCode: existing.statusCode, body: existing.body }

  // Process the request
  const result = await handler()

  // Store the result for future duplicate requests
  await idempotencyStore.set(key, {
    statusCode: result.statusCode,
    body: result.body,
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
  })

  return result
}
```

### Rate Limiting

```typescript
// Rate limit response headers — always include these
// X-RateLimit-Limit: 100          (requests per window)
// X-RateLimit-Remaining: 43       (remaining in current window)
// X-RateLimit-Reset: 1635724800   (when the window resets, Unix timestamp)
// Retry-After: 30                 (seconds to wait, only on 429)

interface RateLimitConfig {
  windowMs: number // Time window in milliseconds
  maxRequests: number // Max requests per window
  keyGenerator: (req: Request) => string // How to identify the client
}

// Different limits for different endpoints
const rateLimits = {
  default: { windowMs: 60_000, maxRequests: 100 },
  auth: { windowMs: 900_000, maxRequests: 10 }, // 10 attempts per 15 min
  search: { windowMs: 60_000, maxRequests: 30 },
  upload: { windowMs: 3600_000, maxRequests: 50 },
  webhook: { windowMs: 60_000, maxRequests: 1000 }, // Higher for webhooks
}
```

### Versioning

```typescript
// URL versioning — simplest, most explicit, recommended
// /api/v1/orders
// /api/v2/orders

// Rules:
// 1. Support current version (N) and previous version (N-1)
// 2. Deprecate with headers, not removal
// 3. Give consumers at least 6 months to migrate
// 4. Only bump major version for breaking changes

// Deprecation headers
// Deprecation: true
// Sunset: Sat, 01 Sep 2025 00:00:00 GMT
// Link: </api/v2/orders>; rel="successor-version"

// Version coexistence — route to different handlers
// routes/v1/orders.ts
// routes/v2/orders.ts
// Both can share the same service layer, only the HTTP contract changes
```

### HATEOAS (Hypermedia)

```typescript
// Include links to related actions/resources — consumer doesn't hardcode URLs
interface OrderDto {
  id: string
  status: string
  total: MoneyDto
  _links: {
    self: { href: string }
    items: { href: string }
    cancel?: { href: string; method: 'POST' }     // Only if cancellable
    pay?: { href: string; method: 'POST' }         // Only if payable
    customer: { href: string }
  }
}

// Example response
{
  "data": {
    "id": "ord_123",
    "status": "placed",
    "total": { "amount": 4999, "currency": "USD", "formatted": "$49.99" },
    "_links": {
      "self": { "href": "/api/orders/ord_123" },
      "items": { "href": "/api/orders/ord_123/items" },
      "cancel": { "href": "/api/orders/ord_123/cancel", "method": "POST" },
      "pay": { "href": "/api/orders/ord_123/pay", "method": "POST" },
      "customer": { "href": "/api/customers/cust_456" }
    }
  }
}
```

---

## 5. API Security Patterns

### Authentication

```typescript
// Bearer token in Authorization header — standard approach
// Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

// API keys for service-to-service communication
// X-API-Key: sk_live_abc123

// Never put tokens in URLs — they get logged
// Bad:  GET /api/orders?token=abc123
// Good: GET /api/orders (with Authorization header)
```

### Authorization — Resource-Level

```typescript
// Always check ownership/permissions at the resource level
async function handleGetOrder(req: Request, res: Response) {
  const order = await orderService.findById(req.params.id)
  if (!order) return res.status(404).json(notFound('Order'))

  // Don't return 403 — it reveals the resource exists
  // Return 404 if the user can't access it
  if (!order.belongsTo(req.user.id) && !req.user.isAdmin) {
    return res.status(404).json(notFound('Order'))
  }

  return res.json(ResponseBuilder.ok(OrderMapper.toDto(order)))
}
```

### Input Sanitization

```typescript
// Limit request body size
app.use(express.json({ limit: '1mb' }))

// Limit string lengths in schemas (already shown with Zod)
// Limit array lengths
// Limit query complexity

// Prevent NoSQL injection — validate types strictly
// Zod handles this: z.string().uuid() won't accept { $gt: "" }

// Rate limit by user, not just IP (users behind NAT share IPs)
```

---

## 6. Internal Service Contracts

### Service Interface with Result Type

```typescript
// Define the contract as an interface — implementation is hidden
interface PaymentService {
  charge(params: ChargeParams): Promise<Result<Charge, PaymentError>>
  refund(chargeId: ChargeId, amount?: Money): Promise<Result<Refund, PaymentError>>
  getPaymentMethods(customerId: CustomerId): Promise<PaymentMethod[]>
}

interface ChargeParams {
  customerId: CustomerId
  amount: Money
  paymentMethodId: string
  idempotencyKey: string // Consumer MUST provide this
  metadata?: Record<string, string>
}

// Typed error union — consumer knows exactly what can go wrong
type PaymentError =
  | { code: 'INSUFFICIENT_FUNDS'; message: string }
  | { code: 'CARD_DECLINED'; declineCode: string; message: string }
  | { code: 'EXPIRED_CARD'; message: string }
  | { code: 'GATEWAY_ERROR'; retryable: boolean; message: string }
```

### CQRS (Command Query Responsibility Segregation)

```typescript
// Separate read and write models when they have different needs

// Commands — write side (validated, domain logic, consistency)
interface CreateOrderCommand {
  customerId: string
  items: Array<{ productId: string; quantity: number }>
  shippingAddress: Address
}

interface CommandHandler<TCommand, TResult = void> {
  execute(command: TCommand): Promise<TResult>
}

class CreateOrderHandler implements CommandHandler<CreateOrderCommand, OrderId> {
  async execute(command: CreateOrderCommand): Promise<OrderId> {
    // Full domain validation, aggregate rules, events
    const order = Order.create(command)
    order.place()
    await this.repository.save(order)
    await this.eventBus.publishAll(order.domainEvents)
    return order.id
  }
}

// Queries — read side (optimized, denormalized, no domain logic)
interface OrderListQuery {
  storeId: string
  status?: OrderStatus
  page: number
  pageSize: number
}

interface QueryHandler<TQuery, TResult> {
  execute(query: TQuery): Promise<TResult>
}

class OrderListHandler implements QueryHandler<OrderListQuery, PaginatedResponse<OrderSummaryDto>> {
  async execute(query: OrderListQuery): Promise<PaginatedResponse<OrderSummaryDto>> {
    // Direct database query, no domain objects, optimized for reading
    const [orders, total] = await this.prisma.$transaction([
      this.prisma.order.findMany({
        where: { storeId: query.storeId, ...(query.status && { status: query.status }) },
        select: {
          id: true,
          status: true,
          total: true,
          createdAt: true,
          _count: { select: { items: true } },
        },
        skip: (query.page - 1) * query.pageSize,
        take: query.pageSize,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.order.count({ where: { storeId: query.storeId } }),
    ])

    return ResponseBuilder.paginated(orders, total, query.page, query.pageSize)
  }
}
```

---

## 7. Anti-Patterns

- **Chatty APIs** — Requiring 5 requests to load one page. Design aggregated endpoints for common views.
- **Leaking internals** — Exposing database column names, auto-increment IDs, or internal enum values. Use DTOs.
- **Inconsistent responses** — Different shapes for different endpoints. Standardize ONE envelope format.
- **Boolean parameters** — `?active=true&deleted=false&archived=false` → use `?status=active`
- **Ignoring idempotency** — POST endpoints that create duplicates on retry. Use idempotency keys.
- **Over-fetching by default** — Returning 50 fields when the list view needs 5. Create summary DTOs.
- **RPC disguised as REST** — `POST /api/getOrders` is not REST. Use `GET /api/orders`.
- **Exposing sequential IDs** — `GET /api/users/1`, `/api/users/2`... Use UUIDs or public IDs.
- **Nested hell** — `/api/stores/1/categories/2/products/3/reviews/4`. Flatten: `/api/reviews/4`.
- **God endpoint** — One endpoint with 30 query parameters for every possible use case. Split by use case.
- **Silently ignoring fields** — If the client sends unknown fields, reject with 400, don't silently ignore.

---

## 8. Process

1. **List the use cases** — What operations does the consumer need? Start from the UI or client.
2. **Identify resources** — Map use cases to nouns (resources) and verbs (HTTP methods).
3. **Define schemas** — Zod schemas for request/response. This is the source of truth.
4. **Create DTOs** — Separate DTOs for create, update, detail, and summary views.
5. **Design for consistency** — Same patterns for naming, errors, pagination across ALL endpoints.
6. **Add security** — Auth, rate limiting, input validation at every endpoint.
7. **Review with consumers** — Show the API surface to the frontend/client team before building.
8. **Document** — OpenAPI spec or TSDoc for every endpoint.

---

## Output

```
## API Design

### Endpoints
| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/orders | List orders | Required |
| POST | /api/orders | Create order | Required |

### Schemas
- `CreateOrderSchema` — [Zod schema with validation rules]
- `ListOrdersSchema` — [query params with defaults]

### DTOs
- `OrderDto` — full detail
- `OrderSummaryDto` — for lists
- `CreateOrderInput` — input

### Error Codes
| Code | Status | When |
|------|--------|------|
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `CONFLICT` | 409 | Duplicate or state conflict |

### Decisions
- [Decision] — [why]
```

## Inspired By

- **RESTful Web APIs** — Leonard Richardson & Mike Amundsen
- **API Design Patterns** — JJ Geewax
- **Build APIs You Won't Hate** — Phil Sturgeon
- **Designing Data-Intensive Applications** — Martin Kleppmann
- **Implementing Domain-Driven Design** — Vaughn Vernon
- **Enterprise Integration Patterns** — Gregor Hohpe & Bobby Woolf
