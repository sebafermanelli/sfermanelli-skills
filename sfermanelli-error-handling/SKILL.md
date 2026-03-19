---
name: sfermanelli-error-handling
description: Design and implement error handling strategies — custom error hierarchies, error boundaries, retry patterns, logging, and graceful degradation. Use this skill when the user asks about error handling, error boundaries, retry logic, error types, or says things like "handle errors properly", "add error handling", "create error types", "implement retries", "graceful degradation", "error boundary". Distinct from debug (which finds existing bugs) — this prevents and handles errors systematically.
---

# Error Handling

Design error handling that makes failures visible, recoverable, and debuggable. Errors are not exceptional — they are expected. A system that handles errors well is more valuable than one that never errors in testing.

## Golden Rule

**Handle errors at the right level.** Don't catch errors you can't meaningfully handle. Don't let errors propagate where they can't be understood. Every catch block should either recover, translate, or report — never swallow.

---

## 1. Error Hierarchy

### Base Error Architecture

```typescript
// Base application error — all custom errors extend this
// The hierarchy separates domain errors from infrastructure errors
abstract class AppError extends Error {
  abstract readonly code: string
  abstract readonly statusCode: number
  abstract readonly isOperational: boolean

  constructor(
    message: string,
    public readonly cause?: Error
  ) {
    super(message)
    this.name = this.constructor.name
    // Maintains proper stack trace in V8 engines
    Error.captureStackTrace(this, this.constructor)
  }

  toJSON() {
    return {
      code: this.code,
      message: this.message,
      ...(process.env.NODE_ENV === 'development' && { stack: this.stack }),
    }
  }
}
```

### Domain Errors

Errors that represent business rule violations. These belong in the domain layer and speak the language of the domain.

```typescript
// Base for all domain errors — business rules that were violated
abstract class DomainError extends AppError {
  readonly isOperational = true
}

// Entity-specific errors — tied to an aggregate
class OrderNotFoundError extends DomainError {
  readonly code = 'ORDER_NOT_FOUND'
  readonly statusCode = 404

  constructor(orderId: string) {
    super(`Order '${orderId}' not found`)
  }
}

class OrderNotModifiableError extends DomainError {
  readonly code = 'ORDER_NOT_MODIFIABLE'
  readonly statusCode = 409

  constructor(orderId: string, currentStatus: string) {
    super(`Order '${orderId}' cannot be modified in '${currentStatus}' status`)
  }
}

class EmptyOrderError extends DomainError {
  readonly code = 'EMPTY_ORDER'
  readonly statusCode = 422

  constructor(orderId: string) {
    super(`Order '${orderId}' must have at least one item to be placed`)
  }
}

class InsufficientStockError extends DomainError {
  readonly code = 'INSUFFICIENT_STOCK'
  readonly statusCode = 409

  constructor(
    public readonly missing: Array<{ productId: string; requested: number; available: number }>
  ) {
    super(`Insufficient stock for ${missing.length} product(s)`)
  }
}

// Cross-domain errors — shared business rules
class InsufficientFundsError extends DomainError {
  readonly code = 'INSUFFICIENT_FUNDS'
  readonly statusCode = 422

  constructor(accountId: string, amount: string) {
    super(`Account '${accountId}' has insufficient funds for ${amount}`)
  }
}

class BusinessRuleViolationError extends DomainError {
  readonly code = 'BUSINESS_RULE_VIOLATION'
  readonly statusCode = 422

  constructor(rule: string, details?: string) {
    super(details ? `${rule}: ${details}` : rule)
  }
}
```

### Application Errors

Errors at the use-case/application layer — validation, auth, permissions.

```typescript
// Validation — input doesn't meet schema requirements
class ValidationError extends AppError {
  readonly code = 'VALIDATION_ERROR'
  readonly statusCode = 400
  readonly isOperational = true

  constructor(
    message: string,
    public readonly fields: Array<{ field: string; message: string; code: string }>
  ) {
    super(message)
  }

  // Factory from Zod errors
  static fromZod(error: z.ZodError): ValidationError {
    return new ValidationError(
      'Invalid request data',
      error.issues.map(issue => ({
        field: issue.path.join('.'),
        message: issue.message,
        code: issue.code.toUpperCase(),
      }))
    )
  }
}

// Authentication — who are you?
class UnauthorizedError extends AppError {
  readonly code: string
  readonly statusCode = 401
  readonly isOperational = true

  constructor(reason: 'MISSING_TOKEN' | 'INVALID_TOKEN' | 'TOKEN_EXPIRED' = 'MISSING_TOKEN') {
    const messages = {
      MISSING_TOKEN: 'Authentication required',
      INVALID_TOKEN: 'Invalid authentication token',
      TOKEN_EXPIRED: 'Authentication token has expired',
    }
    super(messages[reason])
    this.code = reason
  }
}

// Authorization — you are you, but you can't do this
class ForbiddenError extends AppError {
  readonly code = 'FORBIDDEN'
  readonly statusCode = 403
  readonly isOperational = true

  constructor(action: string, resource: string) {
    super(`Not authorized to ${action} on ${resource}`)
  }
}

// Not found — generic, for any resource
class NotFoundError extends AppError {
  readonly code = 'NOT_FOUND'
  readonly statusCode = 404
  readonly isOperational = true

  constructor(resource: string, identifier: string) {
    super(`${resource} '${identifier}' not found`)
  }
}

// Conflict — optimistic locking, duplicate, state conflict
class ConflictError extends AppError {
  readonly code = 'CONFLICT'
  readonly statusCode = 409
  readonly isOperational = true

  constructor(resource: string, reason: string) {
    super(`Conflict on ${resource}: ${reason}`)
  }
}
```

### Infrastructure Errors

Errors from external systems — database, APIs, queues.

```typescript
// Base for infrastructure errors — these may or may not be retryable
abstract class InfrastructureError extends AppError {
  abstract readonly retryable: boolean
}

// Database errors
class DatabaseError extends InfrastructureError {
  readonly code = 'DATABASE_ERROR'
  readonly statusCode = 500
  readonly isOperational = true
  readonly retryable: boolean

  constructor(message: string, cause?: Error, retryable: boolean = false) {
    super(message, cause)
    this.retryable = retryable
  }

  static connectionFailed(cause: Error): DatabaseError {
    return new DatabaseError('Database connection failed', cause, true)
  }

  static queryFailed(query: string, cause: Error): DatabaseError {
    return new DatabaseError(`Query failed: ${query}`, cause, false)
  }

  static timeout(cause: Error): DatabaseError {
    return new DatabaseError('Database query timed out', cause, true)
  }
}

// External service errors
class ExternalServiceError extends InfrastructureError {
  readonly code = 'EXTERNAL_SERVICE_ERROR'
  readonly statusCode = 502
  readonly isOperational = true
  readonly retryable: boolean

  constructor(
    public readonly service: string,
    message: string,
    cause?: Error,
    retryable: boolean = true
  ) {
    super(`${service}: ${message}`, cause)
    this.retryable = retryable
  }

  static timeout(service: string, cause?: Error): ExternalServiceError {
    return new ExternalServiceError(service, 'Request timed out', cause, true)
  }

  static unavailable(service: string, cause?: Error): ExternalServiceError {
    return new ExternalServiceError(service, 'Service unavailable', cause, true)
  }

  static invalidResponse(service: string, cause?: Error): ExternalServiceError {
    return new ExternalServiceError(service, 'Invalid response received', cause, false)
  }
}

// Programming errors — bugs that should never happen in production
class InternalError extends AppError {
  readonly code = 'INTERNAL_ERROR'
  readonly statusCode = 500
  readonly isOperational = false

  constructor(message: string = 'An unexpected error occurred', cause?: Error) {
    super(message, cause)
  }
}
```

### Operational vs Programming Errors

| Operational (expected) | Programming (unexpected)      |
| ---------------------- | ----------------------------- |
| Validation failure     | TypeError, ReferenceError     |
| Resource not found     | Null pointer dereference      |
| Auth failure           | Unhandled promise rejection   |
| Network timeout        | Infinite loop                 |
| Duplicate entry        | Memory leak                   |
| Rate limit exceeded    | Missing environment variable  |
| Insufficient stock     | Assertion failure             |
| External service down  | Wrong type passed to function |

**Key distinction:** Operational errors are part of normal flow — handle them. Programming errors are bugs — fix them, don't handle them.

---

## 2. Result Pattern

### Type-Safe Error Handling Without Exceptions

```typescript
// Result type — forces the caller to handle both cases at compile time
type Result<T, E = AppError> = { ok: true; value: T } | { ok: false; error: E }

// Helper constructors
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value }
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error }
}

// Usage in domain service
class OrderService {
  async createOrder(
    input: CreateOrderInput
  ): Promise<Result<Order, ValidationError | InsufficientStockError>> {
    // Validation
    const validated = this.validator.validate(input)
    if (!validated.ok) return validated

    // Business rule check
    const stock = await this.inventory.check(input.items)
    if (!stock.sufficient) {
      return err(new InsufficientStockError(stock.missing))
    }

    // Happy path
    const order = Order.create(validated.value)
    order.place()
    await this.repository.save(order)
    return ok(order)
  }
}

// Usage in controller — exhaustive handling
async function handleCreateOrder(req: Request, res: Response) {
  const result = await orderService.createOrder(req.body)

  if (!result.ok) {
    const error = result.error
    // TypeScript knows the exact error types
    if (error instanceof ValidationError) {
      return res.status(400).json({ error: { code: error.code, details: error.fields } })
    }
    if (error instanceof InsufficientStockError) {
      return res.status(409).json({ error: { code: error.code, missing: error.missing } })
    }
    // TypeScript would error here if we missed a case (with exhaustive check)
    return assertNever(error)
  }

  return res.status(201).json({ data: OrderMapper.toDto(result.value) })
}
```

### Result Chaining

```typescript
// Chain multiple operations that can fail
class OrderWorkflow {
  async processOrder(
    orderId: string
  ): Promise<Result<OrderConfirmation, OrderError | PaymentError | ShippingError>> {
    // Step 1: Find order
    const orderResult = await this.findOrder(orderId)
    if (!orderResult.ok) return orderResult

    // Step 2: Process payment
    const paymentResult = await this.processPayment(orderResult.value)
    if (!paymentResult.ok) return paymentResult

    // Step 3: Schedule shipping
    const shippingResult = await this.scheduleShipping(orderResult.value, paymentResult.value)
    if (!shippingResult.ok) return shippingResult

    return ok({
      orderId: orderResult.value.id,
      paymentId: paymentResult.value.id,
      trackingNumber: shippingResult.value.trackingNumber,
    })
  }
}
```

### Exhaustive Error Matching

```typescript
// Ensure all error types are handled at compile time
function assertNever(value: never): never {
  throw new InternalError(`Unexpected value: ${JSON.stringify(value)}`)
}

type OrderError = OrderNotFoundError | OrderNotModifiableError | InsufficientStockError

function handleOrderError(error: OrderError): ApiErrorResponse {
  switch (error.code) {
    case 'ORDER_NOT_FOUND':
      return { error: { code: error.code, message: error.message } }
    case 'ORDER_NOT_MODIFIABLE':
      return { error: { code: error.code, message: error.message } }
    case 'INSUFFICIENT_STOCK':
      return { error: { code: error.code, message: error.message, missing: error.missing } }
    default:
      return assertNever(error)
  }
}
```

### When to Use Result vs Throw

| Use Result                                                | Use Throw                               |
| --------------------------------------------------------- | --------------------------------------- |
| Expected failures (validation, not found, business rules) | Unexpected failures (bugs, infra down)  |
| Caller should decide how to handle                        | Failure should propagate up immediately |
| Multiple error types possible                             | One critical failure mode               |
| Domain and application layers                             | Framework boundaries (HTTP handlers)    |
| You want compile-time exhaustiveness checks               | Global error handlers (middleware)      |

---

## 3. Error Boundaries (React)

### Reusable Error Boundary

```tsx
interface ErrorBoundaryState {
  error: Error | null
}

interface ErrorBoundaryProps {
  fallback: React.ComponentType<{ error: Error; reset: () => void }>
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void
  children: React.ReactNode
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { error: null }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    this.props.onError?.(error, errorInfo)
  }

  reset = (): void => {
    this.setState({ error: null })
  }

  render(): React.ReactNode {
    if (this.state.error) {
      const Fallback = this.props.fallback
      return <Fallback error={this.state.error} reset={this.reset} />
    }
    return this.props.children
  }
}
```

### Granular Boundaries — Isolate Failures

```tsx
// Don't wrap the entire app in one boundary — isolate failure zones
function OrderPage() {
  return (
    <Layout>
      {/* Critical — if this fails, show a full-page error */}
      <ErrorBoundary fallback={OrderNotFoundFallback}>
        <OrderHeader />
      </ErrorBoundary>

      {/* Non-critical — if this fails, the rest of the page still works */}
      <ErrorBoundary fallback={SectionErrorFallback}>
        <OrderItems />
      </ErrorBoundary>

      <ErrorBoundary fallback={SectionErrorFallback}>
        <OrderTimeline />
      </ErrorBoundary>

      {/* Independent — recommendations failing shouldn't affect anything */}
      <ErrorBoundary fallback={EmptyFallback}>
        <RelatedOrders />
      </ErrorBoundary>
    </Layout>
  )
}

// Different fallback levels
function SectionErrorFallback({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div role="alert" className="p-4 border rounded bg-red-50">
      <p>Something went wrong loading this section.</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}

function EmptyFallback() {
  return null // Silently hide the broken section
}
```

### Async Error Handling in React

```tsx
// Error boundaries don't catch async errors — handle those in the data layer
function useAsyncOperation<T>() {
  const [state, setState] = useState<{
    data: T | null
    error: AppError | null
    loading: boolean
  }>({ data: null, error: null, loading: false })

  const execute = useCallback(async (operation: () => Promise<T>) => {
    setState({ data: null, error: null, loading: true })
    try {
      const data = await operation()
      setState({ data, error: null, loading: false })
      return data
    } catch (error) {
      const appError =
        error instanceof AppError ? error : new InternalError('Unexpected error', error as Error)
      setState({ data: null, error: appError, loading: false })
      throw appError
    }
  }, [])

  return { ...state, execute }
}

// Usage
function OrderActions({ orderId }: { orderId: string }) {
  const { error, loading, execute } = useAsyncOperation<void>()

  const handleCancel = () => execute(() => orderApi.cancel(orderId))

  if (error) {
    return <ErrorMessage error={error} />
  }

  return (
    <button onClick={handleCancel} disabled={loading}>
      Cancel Order
    </button>
  )
}
```

---

## 4. Retry Patterns

### Exponential Backoff with Jitter

```typescript
interface RetryOptions {
  maxRetries: number
  baseDelayMs: number
  maxDelayMs: number
  retryableErrors?: Array<new (...args: any[]) => Error>
  onRetry?: (attempt: number, error: Error, delayMs: number) => void
}

async function withRetry<T>(operation: () => Promise<T>, options: RetryOptions): Promise<T> {
  const { maxRetries, baseDelayMs, maxDelayMs, retryableErrors, onRetry } = options
  let lastError: Error

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      lastError = error as Error

      // Don't retry non-retryable errors
      if (retryableErrors && !retryableErrors.some(E => error instanceof E)) {
        throw error
      }

      // Don't retry operational domain errors — they won't resolve by retrying
      if (error instanceof DomainError) {
        throw error
      }

      // Don't retry after last attempt
      if (attempt === maxRetries) break

      // Exponential backoff with full jitter
      const exponentialDelay = baseDelayMs * Math.pow(2, attempt)
      const delay = Math.min(Math.random() * exponentialDelay, maxDelayMs)

      onRetry?.(attempt + 1, lastError, delay)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }

  throw lastError!
}

// Usage
const order = await withRetry(() => orderRepository.findById(orderId), {
  maxRetries: 3,
  baseDelayMs: 500,
  maxDelayMs: 5000,
  retryableErrors: [DatabaseError, ExternalServiceError],
  onRetry: (attempt, error, delay) => {
    logger.warn(`Retry attempt ${attempt} after ${delay}ms`, { error: error.message })
  },
})
```

### Circuit Breaker

Prevent cascading failures when an external service is down. Stop calling a failing service and fail fast instead.

```typescript
enum CircuitState {
  Closed = 'CLOSED', // Normal operation — requests pass through
  Open = 'OPEN', // Failing — reject immediately, don't even try
  HalfOpen = 'HALF_OPEN', // Testing — allow one request to test recovery
}

class CircuitBreaker {
  private state = CircuitState.Closed
  private failureCount = 0
  private successCount = 0
  private lastFailureTime = 0

  constructor(
    private readonly name: string,
    private readonly options: {
      failureThreshold: number // Failures before opening (default: 5)
      resetTimeoutMs: number // How long to stay open (default: 30s)
      halfOpenMaxAttempts: number // Successes needed to close from half-open (default: 3)
    } = { failureThreshold: 5, resetTimeoutMs: 30_000, halfOpenMaxAttempts: 3 }
  ) {}

  async execute<T>(operation: () => Promise<T>, fallback?: () => T): Promise<T> {
    if (this.state === CircuitState.Open) {
      if (Date.now() - this.lastFailureTime > this.options.resetTimeoutMs) {
        this.state = CircuitState.HalfOpen
        this.successCount = 0
      } else {
        if (fallback) return fallback()
        throw new CircuitOpenError(this.name, this.options.resetTimeoutMs)
      }
    }

    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      if (fallback && this.state === CircuitState.Open) return fallback()
      throw error
    }
  }

  private onSuccess(): void {
    if (this.state === CircuitState.HalfOpen) {
      this.successCount++
      if (this.successCount >= this.options.halfOpenMaxAttempts) {
        this.state = CircuitState.Closed
        this.failureCount = 0
      }
    } else {
      this.failureCount = 0
    }
  }

  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = Date.now()
    if (this.failureCount >= this.options.failureThreshold) {
      this.state = CircuitState.Open
    }
  }

  getState(): CircuitState {
    return this.state
  }
}

class CircuitOpenError extends InfrastructureError {
  readonly code = 'CIRCUIT_OPEN'
  readonly statusCode = 503
  readonly isOperational = true
  readonly retryable = true

  constructor(service: string, resetTimeoutMs: number) {
    super(`Circuit breaker for '${service}' is open. Retry after ${resetTimeoutMs}ms.`)
  }
}

// Usage — one circuit breaker per external service
const paymentCircuit = new CircuitBreaker('payment-gateway', {
  failureThreshold: 3,
  resetTimeoutMs: 60_000,
  halfOpenMaxAttempts: 2,
})

const charge = await paymentCircuit.execute(
  () => paymentGateway.charge(params),
  () => ({ status: 'queued', message: 'Payment will be processed when service recovers' })
)
```

### Timeout Pattern

```typescript
// Never make a call without a timeout — hanging calls consume resources forever
async function withTimeout<T>(
  operation: Promise<T>,
  timeoutMs: number,
  context: string
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new TimeoutError(context, timeoutMs)), timeoutMs)
  )
  return Promise.race([operation, timeout])
}

class TimeoutError extends InfrastructureError {
  readonly code = 'TIMEOUT'
  readonly statusCode = 504
  readonly isOperational = true
  readonly retryable = true

  constructor(operation: string, timeoutMs: number) {
    super(`Operation '${operation}' timed out after ${timeoutMs}ms`)
  }
}

// Usage
const order = await withTimeout(orderRepository.findById(orderId), 5000, 'findOrderById')
```

### Bulkhead Pattern

Isolate failures so one failing component doesn't consume all resources.

```typescript
// Limit concurrent operations to prevent resource exhaustion
class Bulkhead {
  private running = 0
  private queue: Array<{ resolve: () => void }> = []

  constructor(
    private readonly name: string,
    private readonly maxConcurrent: number,
    private readonly maxQueue: number = 100
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.running >= this.maxConcurrent) {
      if (this.queue.length >= this.maxQueue) {
        throw new BulkheadFullError(this.name)
      }
      await new Promise<void>(resolve => this.queue.push({ resolve }))
    }

    this.running++
    try {
      return await operation()
    } finally {
      this.running--
      const next = this.queue.shift()
      next?.resolve()
    }
  }
}

// Usage — separate bulkheads for separate services
const paymentBulkhead = new Bulkhead('payments', 10, 50)
const notificationBulkhead = new Bulkhead('notifications', 20, 200)

const charge = await paymentBulkhead.execute(() => paymentGateway.charge(params))
```

---

## 5. Logging and Monitoring

### Structured Error Logging

```typescript
interface ErrorLogEntry {
  level: 'error' | 'warn' | 'info'
  code: string
  message: string
  context: Record<string, unknown>
  stack?: string
  timestamp: string
  requestId?: string
  userId?: string
  service?: string
}

class ErrorLogger {
  constructor(
    private readonly service: string,
    private readonly externalReporter?: ExternalErrorReporter // Sentry, Datadog, etc.
  ) {}

  log(error: unknown, context: Record<string, unknown> = {}): void {
    const entry = this.buildEntry(error, context)

    // Operational errors → warn (expected, handled)
    // Programming errors → error (unexpected, needs fix)
    if (entry.level === 'error') {
      console.error(JSON.stringify(entry))
      this.externalReporter?.captureException(error as Error, { extra: context })
    } else {
      console.warn(JSON.stringify(entry))
    }
  }

  private buildEntry(error: unknown, context: Record<string, unknown>): ErrorLogEntry {
    const isApp = error instanceof AppError

    return {
      level: isApp && (error as AppError).isOperational ? 'warn' : 'error',
      code: isApp ? (error as AppError).code : 'UNKNOWN_ERROR',
      message: error instanceof Error ? error.message : String(error),
      context: {
        ...context,
        ...(error instanceof InfrastructureError && { retryable: error.retryable }),
        ...(error instanceof ValidationError && { fields: error.fields }),
        ...(error instanceof ExternalServiceError && { service: error.service }),
      },
      stack: error instanceof Error ? error.stack : undefined,
      timestamp: new Date().toISOString(),
      service: this.service,
    }
  }
}
```

### Request Context — Trace Errors Back to Requests

```typescript
// Attach request ID to every log entry for correlation
import { AsyncLocalStorage } from 'node:async_hooks'

interface RequestContext {
  requestId: string
  userId?: string
  path: string
  method: string
}

const requestStore = new AsyncLocalStorage<RequestContext>()

// Middleware — creates context for each request
function requestContextMiddleware(req: Request, res: Response, next: NextFunction) {
  const context: RequestContext = {
    requestId: (req.headers['x-request-id'] as string) ?? crypto.randomUUID(),
    userId: req.user?.id,
    path: req.path,
    method: req.method,
  }

  res.setHeader('X-Request-Id', context.requestId)

  requestStore.run(context, () => next())
}

// Access context anywhere in the call chain
function getRequestContext(): RequestContext | undefined {
  return requestStore.getStore()
}

// Logger uses context automatically
function logError(error: unknown, extra?: Record<string, unknown>): void {
  const ctx = getRequestContext()
  logger.log(error, {
    ...extra,
    requestId: ctx?.requestId,
    userId: ctx?.userId,
    path: ctx?.path,
  })
}
```

### Global Error Handlers

```typescript
// Last resort — catch anything that wasn't handled
// In Node.js processes:
process.on('unhandledRejection', (reason: unknown) => {
  logError(reason, { handler: 'unhandledRejection' })
  // Graceful shutdown — finish pending requests, then exit
  gracefulShutdown(1)
})

process.on('uncaughtException', (error: Error) => {
  logError(error, { handler: 'uncaughtException' })
  // Must exit — process state is unreliable after uncaught exception
  gracefulShutdown(1)
})

async function gracefulShutdown(exitCode: number): Promise<void> {
  console.log('Starting graceful shutdown...')

  // Stop accepting new requests
  server.close()

  // Wait for pending requests (with timeout)
  await Promise.race([
    waitForPendingRequests(),
    new Promise(resolve => setTimeout(resolve, 10_000)),
  ])

  // Close database connections, flush logs, etc.
  await cleanup()

  process.exit(exitCode)
}
```

---

## 6. HTTP Error Handling

### Centralized Error Middleware

```typescript
// Express-style — all errors funnel through one handler
function errorHandler(error: Error, req: Request, res: Response, next: NextFunction): void {
  // Log with full context
  logError(error, {
    path: req.path,
    method: req.method,
    body: req.body,
    query: req.query,
  })

  // Application errors — return structured response
  if (error instanceof AppError) {
    const response: Record<string, unknown> = {
      error: {
        code: error.code,
        message: error.message,
        requestId: getRequestContext()?.requestId,
      },
    }

    // Add field-level details for validation errors
    if (error instanceof ValidationError) {
      ;(response.error as any).details = error.fields
    }

    // Add retry info for retryable infrastructure errors
    if (error instanceof InfrastructureError && error.retryable) {
      res.setHeader('Retry-After', '30')
    }

    return res.status(error.statusCode).json(response)
  }

  // Unknown errors — don't leak internals
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      requestId: getRequestContext()?.requestId,
    },
  })
}
```

### Async Handler Wrapper

```typescript
// Automatically catches async errors and forwards to error middleware
// Without this, unhandled promise rejections crash the process
function asyncHandler(fn: (req: Request, res: Response, next: NextFunction) => Promise<void>) {
  return (req: Request, res: Response, next: NextFunction) => {
    fn(req, res, next).catch(next)
  }
}

// Usage — no try/catch needed in every handler
router.post(
  '/orders',
  asyncHandler(async (req, res) => {
    const order = await orderService.create(req.body) // Throws? → errorHandler catches it
    res.status(201).json({ data: OrderMapper.toDto(order) })
  })
)

router.get(
  '/orders/:id',
  asyncHandler(async (req, res) => {
    const order = await orderService.findById(req.params.id)
    if (!order) throw new NotFoundError('Order', req.params.id)
    res.json({ data: OrderMapper.toDto(order) })
  })
)
```

### NestJS Exception Filters

```typescript
// NestJS equivalent — exception filter
@Catch()
export class AppExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse<Response>()

    if (exception instanceof AppError) {
      response.status(exception.statusCode).json({
        error: {
          code: exception.code,
          message: exception.message,
          ...(exception instanceof ValidationError && { details: exception.fields }),
        },
      })
      return
    }

    if (exception instanceof HttpException) {
      response.status(exception.getStatus()).json({
        error: {
          code: 'HTTP_ERROR',
          message: exception.message,
        },
      })
      return
    }

    response.status(500).json({
      error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred' },
    })
  }
}
```

---

## 7. Graceful Degradation

### Fallback Strategies

```typescript
// Strategy 1: Default value — feature works with sensible defaults
async function getUserPreferences(userId: string): Promise<Preferences> {
  try {
    return await preferencesService.get(userId)
  } catch (error) {
    logError(error, { userId, fallback: 'defaults' })
    return DEFAULT_PREFERENCES
  }
}

// Strategy 2: Cached value — stale data is better than no data
async function getExchangeRate(currency: string): Promise<number> {
  try {
    const rate = await exchangeRateApi.getRate(currency)
    await cache.set(`rate:${currency}`, rate, { ttl: 3600 })
    return rate
  } catch (error) {
    const cached = await cache.get<number>(`rate:${currency}`)
    if (cached !== null) {
      logError(error, { currency, fallback: 'cache', cachedAge: 'unknown' })
      return cached
    }
    throw error // No fallback available — must propagate
  }
}

// Strategy 3: Feature disable — non-critical feature disappears gracefully
async function getRecommendations(userId: string): Promise<Product[]> {
  try {
    return await recommendationEngine.getForUser(userId)
  } catch (error) {
    logError(error, { userId, fallback: 'disabled' })
    return [] // UI shows nothing — user doesn't even know it broke
  }
}

// Strategy 4: Degraded response — return partial data
async function getOrderWithDetails(orderId: string): Promise<OrderDetailDto> {
  const order = await orderRepository.findById(orderId) // Must succeed
  if (!order) throw new NotFoundError('Order', orderId)

  // These can fail independently — return what we can
  const [customer, shipment, invoice] = await Promise.allSettled([
    customerService.findById(order.customerId),
    shippingService.getTracking(order.id),
    invoiceService.getForOrder(order.id),
  ])

  return {
    ...OrderMapper.toDto(order),
    customer: customer.status === 'fulfilled' ? customer.value : null,
    tracking: shipment.status === 'fulfilled' ? shipment.value : null,
    invoice: invoice.status === 'fulfilled' ? invoice.value : null,
    _partial: [customer, shipment, invoice].some(r => r.status === 'rejected'),
  }
}
```

### Dead Letter Queue Pattern

For async operations that fail — don't lose the work.

```typescript
interface DeadLetterEntry<T> {
  id: string
  payload: T
  error: string
  attempts: number
  firstFailedAt: Date
  lastFailedAt: Date
}

class DeadLetterQueue<T> {
  constructor(private storage: DeadLetterStorage<T>) {}

  async send(payload: T, error: Error, attempts: number): Promise<void> {
    await this.storage.save({
      id: crypto.randomUUID(),
      payload,
      error: error.message,
      attempts,
      firstFailedAt: new Date(),
      lastFailedAt: new Date(),
    })
  }

  async processWithRetry(handler: (entry: DeadLetterEntry<T>) => Promise<void>): Promise<void> {
    const entries = await this.storage.getPending()
    for (const entry of entries) {
      try {
        await handler(entry)
        await this.storage.markProcessed(entry.id)
      } catch (error) {
        await this.storage.updateAttempt(entry.id, (error as Error).message)
      }
    }
  }
}
```

---

## 8. Anti-Patterns

- **Swallowing errors** — Empty catch blocks hide bugs. If you catch it, handle it.
- **Catching too broadly** — `catch (error) { log(error) }` at every level. Handle at the right level only.
- **Throwing strings** — `throw "something failed"`. Always throw Error objects with stack traces.
- **Error as control flow** — Using try/catch for expected branching. Use if/else or Result type.
- **Logging and throwing** — Log OR rethrow, not both. Double-logging obscures the real issue.
- **Generic messages** — "Something went wrong" tells nobody anything. Be specific about what failed.
- **Retrying non-retryable errors** — Retrying a validation error or 404 is pointless. Only retry transient failures.
- **No timeout** — Every external call needs a timeout. A hanging call consumes resources forever.
- **Catching in loops** — `for (item of items) { try { process(item) } catch {} }` silently skips failures.
- **Wrapping every line in try/catch** — Only catch at boundaries where you can meaningfully handle the error.

---

## 9. Process

1. **Classify errors** — Domain, application, or infrastructure? Operational or programming?
2. **Design hierarchy** — Base classes per layer, specific errors per domain concept
3. **Choose handling strategy** — Result for domain, throw for infrastructure, fallback for non-critical
4. **Add boundaries** — Error boundaries in React, middleware in APIs, global handlers as last resort
5. **Implement resilience** — Retries for transient, circuit breaker for external services, timeouts for everything
6. **Add logging** — Structured, with request context, at the right level (warn for operational, error for programming)
7. **Test failure paths** — Every error case should have a test, including retry and fallback behavior

---

## Output

```
## Error Handling Summary

### Error Classes Created
- `OrderNotFoundError` — domain, 404, operational
- `InsufficientStockError` — domain, 409, operational
- `PaymentGatewayError` — infrastructure, 502, retryable

### Boundaries Added
- Error boundary around OrderDetails component
- NestJS exception filter for all AppError subclasses
- Centralized error middleware with request context

### Resilience
- Payment processing: circuit breaker (threshold: 3, reset: 60s)
- External APIs: 3 retries, exponential backoff, 5s timeout
- Notifications: bulkhead (max 20 concurrent)

### Fallback Strategies
- Recommendations: empty array on failure
- Exchange rates: cached value on API timeout
- Order details: partial response with _partial flag

### Verified
- All error paths have tests
- No swallowed errors
- Structured logging with request ID correlation
```

## Inspired By

- **Release It!** — Michael Nygard (Circuit Breaker, Bulkhead, Timeout, Steady State)
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **Node.js Error Handling Best Practices** — Joyent
- **Effective TypeScript** — Dan Vanderkam
- **Domain-Driven Design** — Eric Evans
- **Patterns of Enterprise Application Architecture** — Martin Fowler
- **Enterprise Integration Patterns** — Gregor Hohpe & Bobby Woolf
