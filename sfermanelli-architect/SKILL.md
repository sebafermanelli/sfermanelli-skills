---
name: sfermanelli-architect
description: Design system architecture, folder structure, and high-level decisions for TypeScript projects. Use this skill when the user asks to design a system, plan architecture, structure a project, create an ADR, decide between patterns, or says things like "how should I structure this", "design the architecture", "what pattern should I use", "plan this feature", "write an ADR". Distinct from refactor (which restructures existing code) — this designs new structure from requirements.
---

# Architect

Design systems that are simple, modular, and easy to change. Good architecture makes the right things easy and the wrong things hard. Every architectural decision is a trade-off — document the trade-off, not just the choice.

## Golden Rule

**Complexity is the enemy.** Every layer, abstraction, and indirection must earn its place. The best architecture is the simplest one that meets current requirements and doesn't block foreseeable changes.

---

## 1. Core Principles

### Deep Modules (Ousterhout)

A module should provide a **simple interface** that hides **significant complexity**. If the interface is as complex as the implementation, the module isn't pulling its weight.

```typescript
// Shallow module — interface exposes all complexity
class ImageProcessor {
  loadFromBuffer(buffer: Buffer, format: ImageFormat): void
  decodePixels(colorSpace: ColorSpace): PixelArray
  applyTransform(matrix: TransformMatrix): void
  encodeToFormat(format: ImageFormat, quality: number): Buffer
}

// Deep module — simple interface, complex internals
class ImageProcessor {
  async optimize(input: Buffer, options?: OptimizeOptions): Promise<Buffer>
}
```

### SOLID Principles

**Single Responsibility** — A class should have only one reason to change.

```typescript
// Bad — handles validation, persistence, and notifications
class UserService {
  register(data: RegisterInput) {
    this.validate(data)
    this.saveToDb(data)
    this.sendWelcomeEmail(data)
  }
}

// Good — each concern is separate
class UserRegistration {
  constructor(
    private validator: UserValidator,
    private repository: UserRepository,
    private events: EventEmitter
  ) {}

  async register(data: RegisterInput): Promise<User> {
    const validated = this.validator.validate(data)
    const user = await this.repository.save(validated)
    this.events.emit('user.registered', user)
    return user
  }
}
```

**Open/Closed** — Open for extension, closed for modification. Add behavior through new code, not changing existing code.

```typescript
// Bad — must modify this function for every new discount type
function calculateDiscount(order: Order): number {
  if (order.type === 'wholesale') return order.total * 0.2
  if (order.type === 'employee') return order.total * 0.3
  if (order.type === 'vip') return order.total * 0.15
  return 0
}

// Good — extend by adding new classes
interface DiscountPolicy {
  calculate(order: Order): Money
}

class WholesaleDiscount implements DiscountPolicy {
  calculate(order: Order): Money {
    return order.total.multiply(0.2)
  }
}

class EmployeeDiscount implements DiscountPolicy {
  calculate(order: Order): Money {
    return order.total.multiply(0.3)
  }
}
```

**Liskov Substitution** — Subtypes must be substitutable for their base types without breaking behavior.

```typescript
// Bad — Square violates Rectangle's contract
class Rectangle {
  constructor(
    protected width: number,
    protected height: number
  ) {}
  setWidth(w: number) {
    this.width = w
  }
  setHeight(h: number) {
    this.height = h
  }
  area(): number {
    return this.width * this.height
  }
}

class Square extends Rectangle {
  setWidth(w: number) {
    this.width = w
    this.height = w
  } // breaks expectations
  setHeight(h: number) {
    this.width = h
    this.height = h
  }
}

// Good — model the actual invariant
interface Shape {
  area(): number
}

class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number
  ) {}
  area(): number {
    return this.width * this.height
  }
}

class Square implements Shape {
  constructor(private side: number) {}
  area(): number {
    return this.side * this.side
  }
}
```

**Interface Segregation** — Clients should not depend on interfaces they don't use.

```typescript
// Bad — one fat interface
interface Repository<T> {
  findById(id: string): Promise<T | null>
  findAll(): Promise<T[]>
  save(entity: T): Promise<void>
  delete(id: string): Promise<void>
  softDelete(id: string): Promise<void>
  restore(id: string): Promise<void>
  bulkInsert(entities: T[]): Promise<void>
}

// Good — composed from focused interfaces
interface ReadRepository<T> {
  findById(id: string): Promise<T | null>
  findAll(): Promise<T[]>
}

interface WriteRepository<T> {
  save(entity: T): Promise<void>
  delete(id: string): Promise<void>
}

interface OrderRepository extends ReadRepository<Order>, WriteRepository<Order> {
  findByStore(storeId: string): Promise<Order[]>
}
```

**Dependency Inversion** — High-level modules should not depend on low-level modules. Both should depend on abstractions.

```typescript
// Bad — business logic depends on specific database
class OrderService {
  constructor(private db: PrismaClient) {}
}

// Good — depends on abstraction
interface OrderRepository {
  findById(id: string): Promise<Order | null>
  save(order: Order): Promise<void>
}

class OrderService {
  constructor(private orders: OrderRepository) {}
}

// The implementation lives in infrastructure
class PrismaOrderRepository implements OrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({ where: { id } })
    return data ? OrderMapper.toDomain(data) : null
  }
}
```

---

## 2. Domain-Driven Design (DDD)

### Strategic DDD

#### Bounded Contexts

A Bounded Context is a boundary within which a particular domain model is defined and applicable. The same word can mean different things in different contexts.

```
Example: E-commerce system

[Catalog Context]          [Order Context]           [Shipping Context]
  Product                    Product                   Package
    - name                     - productId               - items[]
    - description              - price                   - weight
    - categories               - quantity                - dimensions
    - images                   - discount                - destination

"Product" means something different in each context.
Each context has its own model, its own rules, its own database (ideally).
```

#### Context Map — Relationships Between Contexts

| Pattern                   | Description                                           | When to use                                 |
| ------------------------- | ----------------------------------------------------- | ------------------------------------------- |
| **Shared Kernel**         | Two contexts share a small common model               | Teams that collaborate closely              |
| **Customer-Supplier**     | Upstream context provides what downstream needs       | Clear provider/consumer relationship        |
| **Anti-Corruption Layer** | Translator between contexts                           | Integrating with legacy or external systems |
| **Published Language**    | Shared protocol/schema (OpenAPI, events)              | Public APIs, event-driven systems           |
| **Separate Ways**         | No integration — each context solves it independently | When integration cost > duplication cost    |

#### Anti-Corruption Layer (ACL) Example

```typescript
// External payment gateway has its own model — don't let it leak into your domain
// infrastructure/payment/stripe-adapter.ts
class StripePaymentAdapter implements PaymentGateway {
  constructor(private stripe: Stripe) {}

  async charge(params: ChargeParams): Promise<ChargeResult> {
    // Translate domain language → Stripe language
    const stripeCharge = await this.stripe.charges.create({
      amount: params.amount.cents,
      currency: params.amount.currency,
      source: params.paymentMethodId,
      metadata: { orderId: params.orderId.value },
    })

    // Translate Stripe language → domain language
    return {
      id: ChargeId.from(stripeCharge.id),
      status: this.mapStatus(stripeCharge.status),
      amount: Money.fromCents(stripeCharge.amount, stripeCharge.currency),
    }
  }

  private mapStatus(stripeStatus: string): ChargeStatus {
    const map: Record<string, ChargeStatus> = {
      succeeded: ChargeStatus.Completed,
      pending: ChargeStatus.Pending,
      failed: ChargeStatus.Failed,
    }
    return map[stripeStatus] ?? ChargeStatus.Unknown
  }
}
```

### Tactical DDD

#### Value Objects

Immutable objects defined by their attributes, not identity. Two value objects with the same attributes are equal.

```typescript
class Money {
  private constructor(
    readonly cents: number,
    readonly currency: string
  ) {
    if (cents < 0) throw new Error('Money cannot be negative')
    if (!currency) throw new Error('Currency is required')
  }

  static fromCents(cents: number, currency: string): Money {
    return new Money(Math.round(cents), currency)
  }

  static fromUnits(units: number, currency: string): Money {
    return new Money(Math.round(units * 100), currency)
  }

  static zero(currency: string = 'USD'): Money {
    return new Money(0, currency)
  }

  add(other: Money): Money {
    this.assertSameCurrency(other)
    return new Money(this.cents + other.cents, this.currency)
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other)
    return new Money(this.cents - other.cents, this.currency)
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.cents * factor), this.currency)
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other)
    return this.cents > other.cents
  }

  equals(other: Money): boolean {
    return this.cents === other.cents && this.currency === other.currency
  }

  format(): string {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: this.currency,
    }).format(this.cents / 100)
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new Error(`Cannot operate on ${this.currency} and ${other.currency}`)
    }
  }
}

class Email {
  readonly value: string

  constructor(value: string) {
    const normalized = value.trim().toLowerCase()
    if (!Email.isValid(normalized)) {
      throw new Error(`Invalid email: ${value}`)
    }
    this.value = normalized
  }

  private static isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
  }

  equals(other: Email): boolean {
    return this.value === other.value
  }

  get domain(): string {
    return this.value.split('@')[1]
  }
}

class DateRange {
  constructor(
    readonly start: Date,
    readonly end: Date
  ) {
    if (end <= start) throw new Error('End must be after start')
  }

  contains(date: Date): boolean {
    return date >= this.start && date <= this.end
  }

  overlaps(other: DateRange): boolean {
    return this.start < other.end && this.end > other.start
  }

  get durationInDays(): number {
    return Math.ceil((this.end.getTime() - this.start.getTime()) / (1000 * 60 * 60 * 24))
  }

  equals(other: DateRange): boolean {
    return (
      this.start.getTime() === other.start.getTime() && this.end.getTime() === other.end.getTime()
    )
  }
}
```

#### Entities

Objects defined by their identity, not their attributes. Two entities with the same ID are the same entity even if other attributes differ.

```typescript
abstract class Entity<TId> {
  constructor(readonly id: TId) {}

  equals(other: Entity<TId>): boolean {
    return this.id === other.id
  }
}

class OrderId {
  constructor(readonly value: string) {
    if (!value) throw new Error('OrderId cannot be empty')
  }

  static generate(): OrderId {
    return new OrderId(crypto.randomUUID())
  }

  equals(other: OrderId): boolean {
    return this.value === other.value
  }

  toString(): string {
    return this.value
  }
}

class Order extends Entity<OrderId> {
  private _status: OrderStatus
  private _items: OrderItem[]
  private _placedAt?: Date

  constructor(
    id: OrderId,
    private readonly customerId: CustomerId,
    items: OrderItem[],
    status: OrderStatus = OrderStatus.Draft
  ) {
    super(id)
    this._items = [...items]
    this._status = status
  }

  get status(): OrderStatus {
    return this._status
  }
  get items(): ReadonlyArray<OrderItem> {
    return this._items
  }
  get placedAt(): Date | undefined {
    return this._placedAt
  }

  get total(): Money {
    return this._items.reduce((sum, item) => sum.add(item.subtotal), Money.zero())
  }

  addItem(product: Product, quantity: number): void {
    this.assertDraft()
    const existing = this._items.find(i => i.productId.equals(product.id))
    if (existing) {
      existing.increaseQuantity(quantity)
    } else {
      this._items.push(OrderItem.create(product, quantity))
    }
  }

  removeItem(productId: ProductId): void {
    this.assertDraft()
    this._items = this._items.filter(i => !i.productId.equals(productId))
  }

  place(): void {
    this.assertDraft()
    if (this._items.length === 0) {
      throw new EmptyOrderError(this.id)
    }
    this._status = OrderStatus.Placed
    this._placedAt = new Date()
  }

  cancel(reason: string): void {
    if (this._status === OrderStatus.Shipped) {
      throw new OrderAlreadyShippedError(this.id)
    }
    this._status = OrderStatus.Cancelled
  }

  private assertDraft(): void {
    if (this._status !== OrderStatus.Draft) {
      throw new OrderNotModifiableError(this.id, this._status)
    }
  }
}
```

#### Aggregates

A cluster of entities and value objects treated as a single unit. The **Aggregate Root** is the only entry point — external objects can only reference the root.

```typescript
// Order is the Aggregate Root
// OrderItem is an entity within the aggregate — no direct access from outside
// Money, OrderId are value objects

// Rules:
// 1. Reference other aggregates by ID only, not by object reference
// 2. One transaction = one aggregate. Don't modify multiple aggregates in one transaction.
// 3. Keep aggregates small. If the aggregate is too big, you drew the boundary wrong.

class Order /* Aggregate Root */ {
  // ✓ References Customer by ID (another aggregate)
  constructor(
    id: OrderId,
    private readonly customerId: CustomerId, // ID reference, not Customer object
    private items: OrderItem[]               // Owned by this aggregate
  ) {}

  // ✓ All mutations go through the root
  addItem(product: Product, quantity: number): void { ... }
  removeItem(productId: ProductId): void { ... }
  place(): void { ... }

  // ✗ Never expose mutable internals
  // Bad: getItems(): OrderItem[] — caller could modify the array
  // Good: readonly access
  get items(): ReadonlyArray<OrderItem> { return this._items }
}
```

#### Domain Events

Something that happened in the domain that other parts of the system care about.

```typescript
interface DomainEvent {
  readonly eventType: string
  readonly occurredAt: Date
  readonly aggregateId: string
}

class OrderPlaced implements DomainEvent {
  readonly eventType = 'order.placed'
  readonly occurredAt = new Date()

  constructor(
    readonly aggregateId: string,
    readonly customerId: string,
    readonly total: { cents: number; currency: string },
    readonly itemCount: number
  ) {}
}

class OrderCancelled implements DomainEvent {
  readonly eventType = 'order.cancelled'
  readonly occurredAt = new Date()

  constructor(
    readonly aggregateId: string,
    readonly reason: string
  ) {}
}

// Aggregate collects events — publish after persistence
abstract class AggregateRoot<TId> extends Entity<TId> {
  private _domainEvents: DomainEvent[] = []

  get domainEvents(): ReadonlyArray<DomainEvent> {
    return this._domainEvents
  }

  protected addEvent(event: DomainEvent): void {
    this._domainEvents.push(event)
  }

  clearEvents(): void {
    this._domainEvents = []
  }
}

class Order extends AggregateRoot<OrderId> {
  place(): void {
    this.assertDraft()
    if (this._items.length === 0) throw new EmptyOrderError(this.id)
    this._status = OrderStatus.Placed
    this._placedAt = new Date()

    this.addEvent(
      new OrderPlaced(
        this.id.value,
        this.customerId.value,
        { cents: this.total.cents, currency: this.total.currency },
        this._items.length
      )
    )
  }
}

// In the application layer — persist then publish
class PlaceOrderUseCase {
  constructor(
    private orders: OrderRepository,
    private eventBus: EventBus
  ) {}

  async execute(orderId: string): Promise<void> {
    const order = await this.orders.findById(OrderId.from(orderId))
    if (!order) throw new OrderNotFoundError(orderId)

    order.place()

    await this.orders.save(order)
    await this.eventBus.publishAll(order.domainEvents)
    order.clearEvents()
  }
}
```

#### Domain Services

Logic that doesn't naturally belong to any entity or value object.

```typescript
// Domain service — business rule that spans multiple aggregates
class TransferService {
  transfer(from: Account, to: Account, amount: Money): void {
    if (!from.canWithdraw(amount)) {
      throw new InsufficientFundsError(from.id, amount)
    }
    from.withdraw(amount)
    to.deposit(amount)
  }
}

// Application service — orchestrates use case (NOT business logic)
class TransferMoneyUseCase {
  constructor(
    private accounts: AccountRepository,
    private transferService: TransferService,
    private eventBus: EventBus
  ) {}

  async execute(input: TransferInput): Promise<void> {
    const from = await this.accounts.findById(input.fromAccountId)
    const to = await this.accounts.findById(input.toAccountId)
    const amount = Money.fromCents(input.amountCents, input.currency)

    this.transferService.transfer(from, to, amount)

    await this.accounts.save(from)
    await this.accounts.save(to)
    await this.eventBus.publishAll([...from.domainEvents, ...to.domainEvents])
  }
}
```

#### Repositories

Collection-like interface for accessing aggregates. The repository interface belongs to the **domain** layer. The implementation belongs to **infrastructure**.

```typescript
// domain/repositories/order-repository.ts
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>
  findByCustomer(customerId: CustomerId): Promise<Order[]>
  save(order: Order): Promise<void>
  delete(id: OrderId): Promise<void>
  nextId(): OrderId
}

// infrastructure/persistence/prisma-order-repository.ts
class PrismaOrderRepository implements OrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { items: true },
    })
    return data ? OrderMapper.toDomain(data) : null
  }

  async save(order: Order): Promise<void> {
    const data = OrderMapper.toPersistence(order)
    await this.prisma.order.upsert({
      where: { id: data.id },
      create: data,
      update: data,
    })
  }

  nextId(): OrderId {
    return OrderId.generate()
  }
}

// Mapper — translates between domain and persistence
class OrderMapper {
  static toDomain(raw: PrismaOrder & { items: PrismaOrderItem[] }): Order {
    return new Order(
      new OrderId(raw.id),
      new CustomerId(raw.customerId),
      raw.items.map(OrderItemMapper.toDomain),
      raw.status as OrderStatus
    )
  }

  static toPersistence(order: Order): PrismaOrderCreateInput {
    return {
      id: order.id.value,
      customerId: order.customerId.value,
      status: order.status,
      items: {
        upsert: order.items.map(item => ({
          where: { id: item.id.value },
          create: OrderItemMapper.toPersistence(item),
          update: OrderItemMapper.toPersistence(item),
        })),
      },
    }
  }
}
```

#### Specifications

Encapsulate business rules as reusable, composable objects.

```typescript
interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean
}

class OrderReadyToShip implements Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return (
      order.status === OrderStatus.Placed &&
      order.isPaid &&
      order.items.every(item => item.isInStock)
    )
  }
}

class HighValueOrder implements Specification<Order> {
  constructor(private threshold: Money) {}

  isSatisfiedBy(order: Order): boolean {
    return order.total.isGreaterThan(this.threshold)
  }
}

// Composition
class AndSpecification<T> implements Specification<T> {
  constructor(
    private left: Specification<T>,
    private right: Specification<T>
  ) {}

  isSatisfiedBy(candidate: T): boolean {
    return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate)
  }
}

// Usage
const readyAndHighValue = new AndSpecification(
  new OrderReadyToShip(),
  new HighValueOrder(Money.fromUnits(500, 'USD'))
)

const priorityOrders = orders.filter(o => readyAndHighValue.isSatisfiedBy(o))
```

---

## 3. Architecture Patterns

### Layered Architecture

```
src/
  domain/              # Pure business logic — no framework deps, no I/O
    entities/          # Aggregates, entities, value objects
    services/          # Domain services (cross-entity logic)
    repositories/      # Repository interfaces (NOT implementations)
    events/            # Domain events
    specifications/    # Business rule objects
    errors/            # Domain-specific errors

  application/         # Use cases — orchestrates domain
    commands/          # Write operations (PlaceOrderCommand)
    queries/           # Read operations (GetOrderQuery)
    dtos/              # Input/Output data transfer objects
    services/          # Application services

  infrastructure/      # External concerns — frameworks, databases, APIs
    persistence/       # Repository implementations (Prisma, Supabase, etc.)
    http/              # Controllers, middleware, route definitions
    messaging/         # Queue consumers, event bus implementation
    external/          # Third-party API adapters (ACL)
    config/            # Environment, DI container setup

  shared/              # Cross-cutting concerns
    types/             # Shared TypeScript types
    errors/            # Base error classes
    utils/             # Pure utility functions
```

### Feature-Based (Vertical Slices)

When features are independent and teams own features:

```
src/
  features/
    orders/
      domain/
        order.entity.ts
        order-item.value-object.ts
        order.repository.ts     # Interface
      application/
        place-order.use-case.ts
        get-order.query.ts
      infrastructure/
        prisma-order.repository.ts
        order.controller.ts
      order.module.ts           # DI wiring
    payments/
      domain/
      application/
      infrastructure/
      payment.module.ts
  shared/
    domain/                     # Shared value objects (Money, Email, etc.)
    infrastructure/             # Shared infra (database client, logger)
```

### When to Use Which

| Pattern       | Use When                                                              |
| ------------- | --------------------------------------------------------------------- |
| Layered       | Complex domain logic, multiple entry points (API + queue + cron)      |
| Feature-based | Feature teams, microservice candidates, simpler domains               |
| Hybrid        | Start feature-based, extract shared domain layer when patterns emerge |

---

## 4. Design Patterns

### Creational Patterns

#### Factory Method

Define an interface for creating objects, but let subclasses decide which class to instantiate.

```typescript
// The creator declares the factory method
abstract class NotificationSender {
  // Factory method — subclasses decide which notification to create
  abstract createNotification(data: NotificationData): Notification

  // Template method that uses the factory
  async send(data: NotificationData): Promise<void> {
    const notification = this.createNotification(data)
    await notification.validate()
    await notification.deliver()
  }
}

class EmailNotificationSender extends NotificationSender {
  createNotification(data: NotificationData): Notification {
    return new EmailNotification(data.recipient.email, data.subject, data.body)
  }
}

class SmsNotificationSender extends NotificationSender {
  createNotification(data: NotificationData): Notification {
    return new SmsNotification(data.recipient.phone, data.body)
  }
}

// Usage
function getNotificationSender(channel: Channel): NotificationSender {
  switch (channel) {
    case 'email':
      return new EmailNotificationSender()
    case 'sms':
      return new SmsNotificationSender()
  }
}
```

#### Abstract Factory

Create families of related objects without specifying their concrete classes.

```typescript
// Abstract factory — creates a family of related UI components
interface UIComponentFactory {
  createButton(label: string): Button
  createInput(placeholder: string): Input
  createModal(title: string): Modal
}

class MaterialUIFactory implements UIComponentFactory {
  createButton(label: string) {
    return new MaterialButton(label)
  }
  createInput(placeholder: string) {
    return new MaterialInput(placeholder)
  }
  createModal(title: string) {
    return new MaterialModal(title)
  }
}

class AntDesignFactory implements UIComponentFactory {
  createButton(label: string) {
    return new AntButton(label)
  }
  createInput(placeholder: string) {
    return new AntInput(placeholder)
  }
  createModal(title: string) {
    return new AntModal(title)
  }
}

// Client code works with any factory — doesn't know concrete classes
class FormBuilder {
  constructor(private factory: UIComponentFactory) {}

  buildLoginForm(): Form {
    return {
      email: this.factory.createInput('Enter email'),
      password: this.factory.createInput('Enter password'),
      submit: this.factory.createButton('Login'),
    }
  }
}
```

#### Builder

Construct complex objects step by step.

```typescript
class QueryBuilder<T> {
  private filters: Filter[] = []
  private sortFields: SortField[] = []
  private pagination?: { page: number; pageSize: number }
  private includes: string[] = []

  where(field: keyof T, operator: FilterOperator, value: unknown): this {
    this.filters.push({ field: field as string, operator, value })
    return this
  }

  orderBy(field: keyof T, direction: 'asc' | 'desc' = 'asc'): this {
    this.sortFields.push({ field: field as string, direction })
    return this
  }

  paginate(page: number, pageSize: number): this {
    this.pagination = { page, pageSize }
    return this
  }

  include(...relations: string[]): this {
    this.includes.push(...relations)
    return this
  }

  build(): Query<T> {
    return new Query<T>(this.filters, this.sortFields, this.pagination, this.includes)
  }
}

// Usage — reads fluently
const query = new QueryBuilder<Order>()
  .where('status', 'eq', 'placed')
  .where('total', 'gte', 10000)
  .orderBy('createdAt', 'desc')
  .include('items', 'customer')
  .paginate(1, 20)
  .build()
```

#### Singleton

Ensure a class has only one instance. **Use sparingly — prefer dependency injection.**

```typescript
// Modern TypeScript approach — module-level singleton
// database.ts
let instance: DatabaseConnection | null = null

export function getDatabase(): DatabaseConnection {
  if (!instance) {
    instance = new DatabaseConnection(config)
  }
  return instance
}

// Better approach — let the DI container manage the lifecycle
// In NestJS: @Injectable({ scope: Scope.DEFAULT }) is singleton by default
// In manual DI: create once, inject everywhere
class Container {
  private db = new DatabaseConnection(config)
  private orderRepo = new PrismaOrderRepository(this.db)
  private orderService = new OrderService(this.orderRepo)

  resolve<T>(token: string): T { ... }
}
```

### Structural Patterns

#### Adapter

Convert the interface of a class into another interface clients expect.

```typescript
// Your domain expects this interface
interface Logger {
  info(message: string, context?: Record<string, unknown>): void
  error(message: string, error?: Error, context?: Record<string, unknown>): void
  warn(message: string, context?: Record<string, unknown>): void
}

// Third-party library has a different interface
class PinoLogger {
  child(bindings: object): PinoLogger { ... }
  info(obj: object, msg?: string): void { ... }
  error(obj: object, msg?: string): void { ... }
}

// Adapter bridges the gap
class PinoLoggerAdapter implements Logger {
  constructor(private pino: PinoLogger) {}

  info(message: string, context?: Record<string, unknown>): void {
    this.pino.info(context ?? {}, message)
  }

  error(message: string, error?: Error, context?: Record<string, unknown>): void {
    this.pino.error({ ...context, err: error }, message)
  }

  warn(message: string, context?: Record<string, unknown>): void {
    this.pino.info(context ?? {}, message) // Pino has no warn — map to info
  }
}
```

#### Decorator

Add behavior to objects dynamically without modifying their class.

```typescript
// Base interface
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>
  save(order: Order): Promise<void>
}

// Base implementation
class PrismaOrderRepository implements OrderRepository {
  async findById(id: OrderId): Promise<Order | null> { ... }
  async save(order: Order): Promise<void> { ... }
}

// Decorator: adds caching
class CachedOrderRepository implements OrderRepository {
  constructor(
    private inner: OrderRepository,
    private cache: CacheClient
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    const cached = await this.cache.get(`order:${id.value}`)
    if (cached) return cached

    const order = await this.inner.findById(id)
    if (order) await this.cache.set(`order:${id.value}`, order, { ttl: 300 })
    return order
  }

  async save(order: Order): Promise<void> {
    await this.inner.save(order)
    await this.cache.delete(`order:${id.value}`)
  }
}

// Decorator: adds logging
class LoggedOrderRepository implements OrderRepository {
  constructor(
    private inner: OrderRepository,
    private logger: Logger
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    this.logger.info('Finding order', { orderId: id.value })
    const order = await this.inner.findById(id)
    this.logger.info(order ? 'Order found' : 'Order not found', { orderId: id.value })
    return order
  }

  async save(order: Order): Promise<void> {
    this.logger.info('Saving order', { orderId: order.id.value })
    await this.inner.save(order)
  }
}

// Compose decorators — order matters
const repository: OrderRepository = new LoggedOrderRepository(
  new CachedOrderRepository(
    new PrismaOrderRepository(prisma),
    cache
  ),
  logger
)
```

#### Facade

Provide a simplified interface to a complex subsystem.

```typescript
// Complex subsystem with many moving parts
class OrderFacade {
  constructor(
    private inventory: InventoryService,
    private pricing: PricingService,
    private payment: PaymentGateway,
    private shipping: ShippingService,
    private notifications: NotificationService
  ) {}

  // Simple interface for the complex "place order" workflow
  async placeOrder(input: PlaceOrderInput): Promise<OrderConfirmation> {
    const availability = await this.inventory.check(input.items)
    if (!availability.allAvailable) {
      throw new ItemsUnavailableError(availability.unavailable)
    }

    const total = await this.pricing.calculate(input.items, input.discountCode)
    const charge = await this.payment.charge(input.paymentMethod, total)
    const shipment = await this.shipping.schedule(input.address, input.items)

    await this.notifications.send(input.customerId, {
      type: 'order_confirmation',
      orderId: charge.orderId,
      trackingNumber: shipment.trackingNumber,
    })

    return { orderId: charge.orderId, trackingNumber: shipment.trackingNumber }
  }
}
```

#### Proxy

Control access to an object.

```typescript
// Lazy-loading proxy — only loads when needed
class LazyOrderDetails implements OrderDetails {
  private loaded: FullOrderDetails | null = null

  constructor(
    private orderId: OrderId,
    private loader: OrderDetailsLoader
  ) {}

  private async ensureLoaded(): Promise<FullOrderDetails> {
    if (!this.loaded) {
      this.loaded = await this.loader.load(this.orderId)
    }
    return this.loaded
  }

  async getItems(): Promise<OrderItem[]> {
    const details = await this.ensureLoaded()
    return details.items
  }

  async getShippingInfo(): Promise<ShippingInfo> {
    const details = await this.ensureLoaded()
    return details.shipping
  }
}
```

### Behavioral Patterns

#### Strategy

Define a family of algorithms, encapsulate each one, and make them interchangeable.

```typescript
interface ShippingCalculator {
  calculate(items: OrderItem[], destination: Address): Money
}

class StandardShipping implements ShippingCalculator {
  calculate(items: OrderItem[], destination: Address): Money {
    const weight = items.reduce((sum, item) => sum + item.weight, 0)
    return Money.fromUnits(weight * 0.5, 'USD')
  }
}

class ExpressShipping implements ShippingCalculator {
  calculate(items: OrderItem[], destination: Address): Money {
    const weight = items.reduce((sum, item) => sum + item.weight, 0)
    return Money.fromUnits(weight * 1.5 + 10, 'USD')
  }
}

class FreeShipping implements ShippingCalculator {
  calculate(): Money {
    return Money.zero()
  }
}

// Context — uses a strategy
class CheckoutService {
  constructor(private shippingCalculator: ShippingCalculator) {}

  calculateTotal(items: OrderItem[], destination: Address): Money {
    const subtotal = items.reduce((sum, i) => sum.add(i.subtotal), Money.zero())
    const shipping = this.shippingCalculator.calculate(items, destination)
    return subtotal.add(shipping)
  }
}
```

#### Observer

Define a one-to-many dependency so that when one object changes state, all its dependents are notified.

```typescript
// Type-safe event emitter
type EventMap = {
  'order.placed': OrderPlaced
  'order.cancelled': OrderCancelled
  'payment.received': PaymentReceived
}

class TypedEventBus {
  private handlers = new Map<string, Array<(event: any) => Promise<void>>>()

  on<K extends keyof EventMap>(
    event: K,
    handler: (payload: EventMap[K]) => Promise<void>
  ): void {
    const existing = this.handlers.get(event as string) ?? []
    this.handlers.set(event as string, [...existing, handler])
  }

  async emit<K extends keyof EventMap>(event: K, payload: EventMap[K]): Promise<void> {
    const handlers = this.handlers.get(event as string) ?? []
    await Promise.allSettled(handlers.map(h => h(payload)))
  }
}

// Subscribers — decoupled from each other
class InventorySubscriber {
  constructor(eventBus: TypedEventBus) {
    eventBus.on('order.placed', this.reserveStock.bind(this))
    eventBus.on('order.cancelled', this.releaseStock.bind(this))
  }

  private async reserveStock(event: OrderPlaced): Promise<void> { ... }
  private async releaseStock(event: OrderCancelled): Promise<void> { ... }
}

class NotificationSubscriber {
  constructor(eventBus: TypedEventBus) {
    eventBus.on('order.placed', this.sendConfirmation.bind(this))
    eventBus.on('payment.received', this.sendReceipt.bind(this))
  }

  private async sendConfirmation(event: OrderPlaced): Promise<void> { ... }
  private async sendReceipt(event: PaymentReceived): Promise<void> { ... }
}
```

#### Command

Encapsulate a request as an object, allowing parameterization, queuing, and undo.

```typescript
interface Command<TResult = void> {
  execute(): Promise<TResult>
}

interface UndoableCommand<TResult = void> extends Command<TResult> {
  undo(): Promise<void>
}

class PlaceOrderCommand implements Command<OrderId> {
  constructor(
    private orders: OrderRepository,
    private input: PlaceOrderInput
  ) {}

  async execute(): Promise<OrderId> {
    const order = Order.create(this.input)
    order.place()
    await this.orders.save(order)
    return order.id
  }
}

class ChangeOrderStatusCommand implements UndoableCommand {
  private previousStatus?: OrderStatus

  constructor(
    private orders: OrderRepository,
    private orderId: OrderId,
    private newStatus: OrderStatus
  ) {}

  async execute(): Promise<void> {
    const order = await this.orders.findById(this.orderId)
    if (!order) throw new OrderNotFoundError(this.orderId)
    this.previousStatus = order.status
    order.changeStatus(this.newStatus)
    await this.orders.save(order)
  }

  async undo(): Promise<void> {
    if (!this.previousStatus) return
    const order = await this.orders.findById(this.orderId)
    if (!order) return
    order.changeStatus(this.previousStatus)
    await this.orders.save(order)
  }
}

// Command bus — central dispatch
class CommandBus {
  private handlers = new Map<string, (command: any) => Promise<any>>()

  register<T extends Command>(commandType: string, handler: (command: T) => Promise<any>): void {
    this.handlers.set(commandType, handler)
  }

  async dispatch<TResult>(command: Command<TResult>): Promise<TResult> {
    const handler = this.handlers.get(command.constructor.name)
    if (!handler) throw new Error(`No handler for ${command.constructor.name}`)
    return handler(command)
  }
}
```

#### Chain of Responsibility

Pass a request along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler.

```typescript
interface Middleware<T> {
  handle(request: T, next: () => Promise<T>): Promise<T>
}

class ValidationMiddleware implements Middleware<OrderRequest> {
  async handle(request: OrderRequest, next: () => Promise<OrderRequest>): Promise<OrderRequest> {
    if (!request.items.length) throw new ValidationError('Order must have items')
    return next()
  }
}

class AuthorizationMiddleware implements Middleware<OrderRequest> {
  async handle(request: OrderRequest, next: () => Promise<OrderRequest>): Promise<OrderRequest> {
    if (!request.user.canCreateOrders) throw new ForbiddenError()
    return next()
  }
}

class RateLimitMiddleware implements Middleware<OrderRequest> {
  async handle(request: OrderRequest, next: () => Promise<OrderRequest>): Promise<OrderRequest> {
    if (await this.isRateLimited(request.user.id)) throw new TooManyRequestsError()
    return next()
  }
}

// Pipeline builder
class Pipeline<T> {
  private middlewares: Middleware<T>[] = []

  use(middleware: Middleware<T>): this {
    this.middlewares.push(middleware)
    return this
  }

  async execute(request: T, finalHandler: () => Promise<T>): Promise<T> {
    const chain = this.middlewares.reduceRight(
      (next, middleware) => () => middleware.handle(request, next),
      finalHandler
    )
    return chain()
  }
}

// Usage
const pipeline = new Pipeline<OrderRequest>()
  .use(new RateLimitMiddleware())
  .use(new AuthorizationMiddleware())
  .use(new ValidationMiddleware())

await pipeline.execute(request, () => orderService.create(request))
```

#### State

Allow an object to alter its behavior when its internal state changes.

```typescript
interface OrderState {
  place(order: Order): void
  pay(order: Order): void
  ship(order: Order): void
  cancel(order: Order): void
}

class DraftState implements OrderState {
  place(order: Order): void {
    if (order.items.length === 0) throw new EmptyOrderError(order.id)
    order.transitionTo(new PlacedState())
  }
  pay(): void {
    throw new InvalidTransitionError('Draft', 'Paid')
  }
  ship(): void {
    throw new InvalidTransitionError('Draft', 'Shipped')
  }
  cancel(order: Order): void {
    order.transitionTo(new CancelledState())
  }
}

class PlacedState implements OrderState {
  place(): void {
    throw new InvalidTransitionError('Placed', 'Placed')
  }
  pay(order: Order): void {
    order.transitionTo(new PaidState())
  }
  ship(): void {
    throw new InvalidTransitionError('Placed', 'Shipped')
  }
  cancel(order: Order): void {
    order.transitionTo(new CancelledState())
  }
}

class PaidState implements OrderState {
  place(): void {
    throw new InvalidTransitionError('Paid', 'Placed')
  }
  pay(): void {
    throw new InvalidTransitionError('Paid', 'Paid')
  }
  ship(order: Order): void {
    order.transitionTo(new ShippedState())
  }
  cancel(): void {
    throw new InvalidTransitionError('Paid', 'Cancelled — refund required')
  }
}

class Order {
  private state: OrderState = new DraftState()

  transitionTo(state: OrderState): void {
    this.state = state
  }

  place(): void {
    this.state.place(this)
  }
  pay(): void {
    this.state.pay(this)
  }
  ship(): void {
    this.state.ship(this)
  }
  cancel(): void {
    this.state.cancel(this)
  }
}
```

#### Template Method

Define the skeleton of an algorithm, deferring some steps to subclasses.

```typescript
abstract class DataImporter<TRaw, TEntity> {
  // Template method — defines the algorithm
  async import(source: string): Promise<ImportResult> {
    const raw = await this.extract(source)
    const validated = this.validate(raw)
    const entities = validated.map(item => this.transform(item))
    const result = await this.load(entities)
    await this.notify(result)
    return result
  }

  // Steps that subclasses must implement
  protected abstract extract(source: string): Promise<TRaw[]>
  protected abstract validate(items: TRaw[]): TRaw[]
  protected abstract transform(item: TRaw): TEntity

  // Steps with default behavior — subclasses may override
  protected async load(entities: TEntity[]): Promise<ImportResult> {
    let imported = 0
    for (const batch of chunk(entities, 100)) {
      imported += await this.saveBatch(batch)
    }
    return { imported, total: entities.length }
  }

  protected abstract saveBatch(batch: TEntity[]): Promise<number>

  protected async notify(result: ImportResult): Promise<void> {
    console.log(`Imported ${result.imported}/${result.total} records`)
  }
}

class CsvProductImporter extends DataImporter<CsvRow, Product> {
  protected async extract(source: string): Promise<CsvRow[]> {
    return parseCsv(await readFile(source, 'utf-8'))
  }

  protected validate(rows: CsvRow[]): CsvRow[] {
    return rows.filter(row => row.name && row.price)
  }

  protected transform(row: CsvRow): Product {
    return new Product(row.name, Money.fromUnits(parseFloat(row.price), 'USD'))
  }

  protected async saveBatch(batch: Product[]): Promise<number> {
    await this.prisma.product.createMany({ data: batch.map(ProductMapper.toPersistence) })
    return batch.length
  }
}
```

---

## 5. Architecture Decision Records (ADR)

### Template

```markdown
# ADR-XXX: [Decision Title]

## Status

Proposed | Accepted | Deprecated | Superseded by ADR-YYY

## Context

What is the problem? What constraints exist? What forces are at play?

## Decision

What is the change being proposed or made?

## Consequences

### Positive

- What gets better

### Negative

- What gets worse or more complex

### Trade-offs

- What we're giving up in exchange for what we're gaining

## Alternatives Considered

### Alternative A

- Pros
- Cons
- Why rejected
```

---

## 6. Anti-Patterns

- **Premature abstraction** — Don't create interfaces with a single implementation "just in case". Wait until you have two concrete cases.
- **Architecture astronaut** — Don't add layers that don't solve a current problem. YAGNI.
- **Big bang design** — Don't design everything upfront. Design what you need now, refactor when patterns emerge.
- **Framework coupling** — Don't let framework types leak into your domain. Keep the domain pure.
- **God class** — If a class has 10+ methods or 500+ lines, it's doing too much.
- **Anemic domain model** — If your entities are just data bags with no behavior, you're writing procedural code with classes.
- **Wrong aggregate boundaries** — If you're modifying multiple aggregates in one transaction, your boundaries are wrong.
- **Over-engineering DDD** — Not every CRUD app needs DDD. Use it where the domain is complex enough to justify it.

---

## 7. Process

1. **Understand requirements** — What does the system need to do? What are the constraints?
2. **Identify bounded contexts** — Where are the natural boundaries in the domain?
3. **Choose the simplest pattern** — Start with the least complex architecture that works
4. **Define aggregates and entities** — What are the invariants? Where are the consistency boundaries?
5. **Design interfaces first** — Contracts between modules before implementation
6. **Document trade-offs** — Write an ADR for every non-obvious decision
7. **Validate with use cases** — Walk through 3-5 real scenarios to verify the design

---

## Output

```
## Architecture Design

### Overview
[One paragraph describing the system]

### Bounded Contexts
- [Context] — [Responsibility]

### Structure
[Directory tree or module diagram]

### Key Decisions
- **Decision 1** — [What and why]

### Patterns Used
- [Pattern] — [Where and why]

### Trade-offs
- [What we chose] over [what we rejected] because [reason]

### ADRs Created
- ADR-001: [Title]
```

## Inspired By

- **A Philosophy of Software Design** — John Ousterhout
- **Clean Architecture** — Robert C. Martin
- **Domain-Driven Design** — Eric Evans
- **Implementing Domain-Driven Design** — Vaughn Vernon
- **Patterns of Enterprise Application Architecture** — Martin Fowler
- **Design Patterns: Elements of Reusable OO Software** — Gamma, Helm, Johnson, Vlissides (Gang of Four)
- **Fundamentals of Software Architecture** — Mark Richards & Neal Ford
- **Head First Design Patterns** — Eric Freeman & Elisabeth Robson
- **refactoring.guru/design-patterns** — Alexander Shvets
