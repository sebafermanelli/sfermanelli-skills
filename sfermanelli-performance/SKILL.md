---
name: sfermanelli-performance
description: Identify and fix performance bottlenecks in code, queries, and rendering. Use this skill when the user asks to optimize performance, speed up the app, fix slow queries, reduce bundle size, improve load time, or says things like "this is slow", "why is it taking so long", "optimize this", "the page is laggy", "reduce loading time", "improve speed", "find bottlenecks". Also triggers for N+1 queries, unnecessary re-renders, large bundles, or database optimization.
---

# Performance

Find and fix the bottlenecks that actually matter — not micro-optimizations that save nanoseconds, but changes that users can feel.

## Golden Rule

**Measure before optimizing.** If you can't measure the improvement, you can't prove the optimization worked. Premature optimization is the root of all evil — but ignoring a 3-second page load is negligence.

---

## 1. Performance Audit Process

1. **Identify the symptom** — what's slow? Page load? API response? UI interaction?
2. **Measure the baseline** — how slow is it? Consistently or intermittently?
3. **Find the bottleneck** — profile, don't guess. The bottleneck is rarely where you think.
4. **Fix the bottleneck** — apply the minimum change for maximum impact.
5. **Measure the improvement** — did it get faster? By how much?

---

## 2. Web Vitals

The metrics Google uses to measure user experience. Target these for real-world performance.

| Metric                              | What it measures                         | Good    | Needs Work | Poor    |
| ----------------------------------- | ---------------------------------------- | ------- | ---------- | ------- |
| **LCP** (Largest Contentful Paint)  | When the main content is visible         | < 2.5s  | 2.5–4s     | > 4s    |
| **INP** (Interaction to Next Paint) | Response time to user input              | < 200ms | 200–500ms  | > 500ms |
| **CLS** (Cumulative Layout Shift)   | Visual stability (things jumping around) | < 0.1   | 0.1–0.25   | > 0.25  |
| **FCP** (First Contentful Paint)    | When first content appears               | < 1.8s  | 1.8–3s     | > 3s    |
| **TTFB** (Time to First Byte)       | Server response time                     | < 800ms | 800ms–1.8s | > 1.8s  |

### How to Measure

```bash
# Lighthouse (Chrome DevTools → Lighthouse tab)
# PageSpeed Insights: https://pagespeed.web.dev/
# Web Vitals library in code:

npm install web-vitals
```

```typescript
import { onLCP, onINP, onCLS } from 'web-vitals'

onLCP(console.log) // { name: 'LCP', value: 2100, rating: 'good' }
onINP(console.log) // { name: 'INP', value: 150, rating: 'good' }
onCLS(console.log) // { name: 'CLS', value: 0.05, rating: 'good' }
```

### Common Fixes by Metric

| Metric | Problem                    | Fix                                                 |
| ------ | -------------------------- | --------------------------------------------------- |
| LCP    | Large hero image           | Use `next/image`, preload, serve WebP               |
| LCP    | Slow server response       | Add caching, optimize DB query, use CDN             |
| INP    | Heavy JS on interaction    | Debounce, virtualize lists, code-split              |
| INP    | Layout thrashing           | Batch DOM reads/writes, use `requestAnimationFrame` |
| CLS    | Images without dimensions  | Always set `width`/`height` on `<img>`              |
| CLS    | Dynamic content above fold | Reserve space with skeleton/placeholder             |
| CLS    | Web fonts loading late     | Use `font-display: swap`, preload fonts             |

---

## 3. Database / Queries

### N+1 Queries — The #1 Backend Performance Killer

```typescript
// BAD: 1 query for stores + N queries for reviews (N+1)
const stores = await prisma.store.findMany()
for (const store of stores) {
  const reviews = await prisma.review.findMany({ where: { storeId: store.id } })
  store.reviewCount = reviews.length
}
// If 100 stores → 101 queries!

// GOOD: 1 query with include
const stores = await prisma.store.findMany({
  include: {
    _count: { select: { reviews: true } },
  },
})
// 1 query, done.

// GOOD: With Supabase
const { data: stores } = await supabase.from('stores').select('*, reviews(count)')
```

### EXPLAIN ANALYZE — Read Query Plans

```sql
-- Run this before and after optimization
EXPLAIN ANALYZE
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC
LIMIT 20;

-- What to look for in the output:
-- ✗ Seq Scan on orders        → Missing index! Add one on the WHERE columns
-- ✗ Nested Loop (rows=50000)  → Bad join strategy, missing index on join column
-- ✗ Sort (cost=high)          → Add index on ORDER BY column
-- ✓ Index Scan using idx_x    → Good, using an index
-- ✓ Bitmap Index Scan         → Good, combining multiple indexes

-- Compare actual vs estimated rows
-- If they differ by 10x+, run ANALYZE to update statistics:
ANALYZE orders;
```

### Indexing Strategy

```sql
-- Rule 1: Index columns you WHERE, JOIN, and ORDER BY

-- Single column index — for equality and range queries
CREATE INDEX idx_orders_status ON orders(status);

-- Composite index — column order matters!
-- Put equality columns first, range columns last
CREATE INDEX idx_orders_status_created
  ON orders(status, created_at DESC);
-- This supports: WHERE status = 'pending' ORDER BY created_at DESC
-- But NOT: WHERE created_at > X (status must come first)

-- Partial index — smaller, faster, for common queries
CREATE INDEX idx_orders_pending
  ON orders(created_at DESC)
  WHERE status = 'pending';

-- Covering index — query reads only from the index, never touches the table
CREATE INDEX idx_orders_list
  ON orders(status, created_at DESC)
  INCLUDE (id, total, customer_id);

-- For large tables, create indexes concurrently (no lock)
CREATE INDEX CONCURRENTLY idx_orders_customer
  ON orders(customer_id);

-- Check unused indexes (they slow down writes for nothing)
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Over-fetching

```typescript
// BAD: fetching everything
const orders = await prisma.order.findMany() // All columns, all rows

// GOOD: select only needed columns
const orders = await prisma.order.findMany({
  select: { id: true, status: true, total: true, createdAt: true },
  where: { storeId },
  take: 20,
  orderBy: { createdAt: 'desc' },
})

// BAD: loading deep relations you don't need
const order = await prisma.order.findUnique({
  where: { id },
  include: { items: { include: { product: { include: { category: true } } } } },
})

// GOOD: only include what the UI needs
const order = await prisma.order.findUnique({
  where: { id },
  include: { items: { select: { id: true, productName: true, quantity: true, price: true } } },
})
```

### Parallel Queries

```typescript
// BAD: sequential — total time = sum of all queries
const stores = await getStores() // 200ms
const orders = await getOrders() // 300ms
const analytics = await getAnalytics() // 500ms
// Total: 1000ms

// GOOD: parallel — total time = slowest query
const [stores, orders, analytics] = await Promise.all([
  getStores(), // 200ms
  getOrders(), // 300ms
  getAnalytics(), // 500ms
])
// Total: 500ms

// EVEN BETTER: parallel with error isolation
const [storesResult, ordersResult, analyticsResult] = await Promise.allSettled([
  getStores(),
  getOrders(),
  getAnalytics(), // If this fails, stores and orders still work
])
```

### Connection Pooling

```typescript
// Without pooling: new connection per query (~50ms overhead each)
// With pooling: reuse existing connections

// Prisma — connection pool built-in, configure in DATABASE_URL
// postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=30

// NestJS + TypeORM — configure pool
TypeOrmModule.forRoot({
  type: 'postgres',
  extra: {
    max: 20, // Max connections in pool
    idleTimeoutMillis: 30000,
  },
})
```

---

## 4. React / Next.js Performance

### Unnecessary Re-renders

```typescript
// Diagnose: React DevTools → Profiler → "Highlight updates"

// Problem 1: New object reference every render
<Context.Provider value={{ user, setUser }}>  // New object each render
// Fix:
const value = useMemo(() => ({ user, setUser }), [user])
<Context.Provider value={value}>

// Problem 2: Inline functions as props
<Button onClick={() => handleClick(id)} />  // New function each render
// Fix: useCallback or move handler
const handleButtonClick = useCallback(() => handleClick(id), [id])
<Button onClick={handleButtonClick} />

// Problem 3: Computing derived data in render
function OrderList({ orders }) {
  const sorted = orders.sort((a, b) => b.date - a.date)  // Sorts every render
  // Fix:
  const sorted = useMemo(
    () => [...orders].sort((a, b) => b.date - a.date),
    [orders]
  )
}
```

### Code Splitting and Lazy Loading

```typescript
// Next.js — automatic route-based code splitting
// Each page in /app is its own bundle

// Dynamic imports for heavy components
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/Chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false  // Client-only (e.g., uses canvas/WebGL)
})

// React.lazy for non-Next.js
const AdminPanel = React.lazy(() => import('./AdminPanel'))
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <AdminPanel />
    </Suspense>
  )
}
```

### Bundle Size Analysis

```bash
# Next.js
npx @next/bundle-analyzer
# Or add to next.config.js:
const withBundleAnalyzer = require('@next/bundle-analyzer')({ enabled: true })
module.exports = withBundleAnalyzer(nextConfig)

# Generic
npx source-map-explorer 'build/**/*.js'
```

```typescript
// Common bundle bloaters and fixes

// Bad: importing entire library
import _ from 'lodash'              // 71KB
// Good: import only what you need
import groupBy from 'lodash/groupBy' // 1.5KB

// Bad: importing heavy library synchronously
import { format } from 'date-fns'   // 30KB
// Good: use native Intl or dynamic import
new Intl.DateTimeFormat('en', { dateStyle: 'medium' }).format(date)

// Check for duplicates
npx depcheck  // Find unused dependencies
```

### Image Optimization

```tsx
// BAD: unoptimized image
<img src="/hero.jpg" />

// GOOD: Next.js Image (automatic WebP, lazy loading, responsive)
import Image from 'next/image'
<Image
  src="/hero.jpg"
  width={1200}
  height={600}
  alt="Hero banner"
  priority           // Above the fold → preload
  sizes="(max-width: 768px) 100vw, 1200px"
/>

// For background images or CSS:
// Convert to WebP: cwebp input.png -o output.webp
// Use <picture> for fallback:
<picture>
  <source srcSet="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" alt="Hero" />
</picture>
```

### Server vs Client Components (Next.js)

```
Server Components (default):
  ✓ Data fetching (no client JS shipped)
  ✓ Access to server resources (DB, filesystem)
  ✓ Sensitive data (API keys, tokens)
  ✗ No interactivity (no onClick, no useState)

Client Components ("use client"):
  ✓ Interactivity (events, state, effects)
  ✓ Browser APIs (localStorage, geolocation)
  ✗ Increases bundle size
  ✗ Data fetching causes waterfalls

Rule: Keep client components as small and as low in the tree as possible.
Don't make a whole page "use client" for one button.
```

### Caching (Next.js)

```typescript
// Static data — cache aggressively
export const revalidate = 3600 // Revalidate every hour

// Dynamic data — ISR (Incremental Static Regeneration)
export const revalidate = 60 // Revalidate every minute

// On-demand revalidation — after mutations
import { revalidatePath, revalidateTag } from 'next/cache'

async function createOrder(data: CreateOrderInput) {
  await db.order.create({ data })
  revalidatePath('/orders')
  revalidateTag('orders')
}

// Fetch caching
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600, tags: ['products'] },
})

// Force dynamic (no caching)
export const dynamic = 'force-dynamic'
```

---

## 5. Angular Performance

### Change Detection

```typescript
// Angular checks ALL components by default (zone.js)
// OnPush: only checks when inputs change or events fire

@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
export class OrderListComponent {
  // With OnPush, Angular only re-checks when:
  // 1. @Input() reference changes (not mutation!)
  // 2. Event handler fires (click, etc.)
  // 3. Async pipe emits
  // 4. markForCheck() called manually
}

// BAD: Mutating array (OnPush won't detect)
this.items.push(newItem)

// GOOD: New reference (OnPush detects)
this.items = [...this.items, newItem]
```

### trackBy for ngFor

```typescript
// Without trackBy: Angular destroys and recreates ALL DOM elements on change
@Component({
  template: `
    <div *ngFor="let order of orders; trackBy: trackByOrderId">
      {{ order.name }}
    </div>
  `,
})
export class OrderListComponent {
  trackByOrderId(index: number, order: Order): string {
    return order.id // Angular reuses DOM elements with same ID
  }
}
```

### Lazy Loading Modules/Routes

```typescript
// Lazy load routes — only loads the module when the route is visited
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
  },
  {
    path: 'orders',
    loadComponent: () => import('./orders/order-list.component').then(c => c.OrderListComponent),
  },
]
```

### Observable Optimization

```typescript
// BAD: Multiple subscriptions to the same HTTP call
// Template: {{ orders$ | async }} used in 3 places = 3 HTTP requests

// GOOD: Share the observable
orders$ = this.http.get<Order[]>('/api/orders').pipe(shareReplay({ bufferSize: 1, refCount: true }))

// BAD: Heavy computation in template
// Template: {{ getTotal(orders) }} — called every change detection cycle

// GOOD: Use a pipe or pre-compute
totalOrders$ = this.orders$.pipe(map(orders => orders.reduce((sum, o) => sum + o.total, 0)))
```

---

## 6. NestJS Performance

### Caching

```typescript
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager'

@Module({
  imports: [
    CacheModule.register({
      ttl: 60, // Default TTL in seconds
      max: 100, // Max items in cache
    }),
  ],
})
export class AppModule {}

// Cache specific endpoints
@Controller('products')
export class ProductController {
  @Get()
  @UseInterceptors(CacheInterceptor)
  @CacheTTL(300) // Cache for 5 minutes
  async findAll(): Promise<Product[]> {
    return this.productService.findAll()
  }
}

// Manual cache management
@Injectable()
export class ProductService {
  constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}

  async findById(id: string): Promise<Product> {
    const cached = await this.cache.get<Product>(`product:${id}`)
    if (cached) return cached

    const product = await this.prisma.product.findUnique({ where: { id } })
    await this.cache.set(`product:${id}`, product, 300)
    return product
  }

  async update(id: string, data: UpdateProductDto): Promise<Product> {
    const product = await this.prisma.product.update({ where: { id }, data })
    await this.cache.del(`product:${id}`) // Invalidate cache
    return product
  }
}
```

### Compression

```typescript
import compression from 'compression'

async function bootstrap() {
  const app = await NestFactory.create(AppModule)
  app.use(compression()) // Gzip responses (30-70% size reduction)
  await app.listen(3000)
}
```

---

## 7. Memory Leaks

### Common Patterns

```typescript
// Leak 1: Event listeners not removed
window.addEventListener('resize', handler)
// Fix: remove in cleanup (useEffect return, ngOnDestroy, etc.)

// Leak 2: Timers not cleared
const interval = setInterval(poll, 5000)
// Fix: clearInterval in cleanup

// Leak 3: Closures holding references
function createHandler() {
  const hugeData = loadHugeDataset()  // 50MB in memory
  return () => {
    console.log(hugeData.length)      // Closure keeps hugeData alive forever
  }
}

// Leak 4: Unsubscribed Observables (Angular)
this.service.getData().subscribe(...)  // Never unsubscribed
// Fix: takeUntil(destroy$) or async pipe

// Leak 5: Growing collections without bounds
const cache = new Map()
function processRequest(req) {
  cache.set(req.id, result)  // Cache grows forever
}
// Fix: use LRU cache with max size, or TTL-based expiration

// Detecting leaks:
// Chrome DevTools → Memory tab → Take heap snapshot
// Compare snapshots before and after the suspected leak
// Look for objects that keep growing
```

---

## 8. Caching Patterns

| Pattern                    | How it works                                             | Use when                         |
| -------------------------- | -------------------------------------------------------- | -------------------------------- |
| **Cache-aside**            | App checks cache, loads from DB on miss, writes to cache | Most common. Good default.       |
| **Write-through**          | Write to cache and DB together, read from cache          | Read-heavy, need consistency     |
| **Write-behind**           | Write to cache, async flush to DB                        | Write-heavy, can tolerate async  |
| **TTL-based**              | Set expiration, accept stale data within window          | Data freshness isn't critical    |
| **Stale-while-revalidate** | Return cached, refresh in background                     | Best UX, eventual consistency OK |

### Denormalization Tradeoffs

Normalize for writes, denormalize for reads. If a query joins 5 tables and runs 1000x/day, consider storing the precomputed result.

---

## 9. Performance Report

````markdown
## Performance Audit

### Bottleneck Found

**Symptom:** Store detail page takes 3.2s to load
**Root cause:** Sequential queries — getStoreInfo, getProducts, getReviews run one after another
**Impact:** ~2s wasted waiting

### Fix Applied

Parallelized with `Promise.all` and added missing index:

```typescript
const [store, products, reviews] = await Promise.all([
  getStoreInfo(slug),
  getProducts(storeId),
  getReviews(storeId),
])
```
````

### Result

- Before: 3.2s
- After: 1.1s
- Improvement: 66% faster

### Web Vitals Impact

- LCP: 3.8s → 1.6s (good)
- INP: 280ms → 120ms (good)
- CLS: 0.15 → 0.02 (good)

### Additional Recommendations

1. Add index on `products.store_id` (currently missing)
2. Consider caching store info with `revalidate = 60`
3. Lazy load review section below the fold

```

---

## 10. Principles

- **Optimize for the user, not the machine** — 100ms improvement on a 200ms operation is invisible. 1s improvement on a 4s operation is transformative.
- **The fastest code is code that doesn't run** — can you cache it? Can you avoid the computation entirely?
- **Measure in production conditions** — dev mode is not representative. SQLite is not Postgres.
- **Network is usually the bottleneck** — reduce round trips before optimizing CPU.
- **Don't sacrifice readability for speed** — unless the performance gain is significant and measured.

### Data Layer (Kleppmann — Designing Data-Intensive Applications)

- **B-tree indexes** (default) — great for equality and range queries
- **Composite indexes** — column order matters: equality first, range last
- **Cover your queries** — if a query only needs indexed columns, it never touches the table

### Network (Grigorik — High Performance Browser Networking)

- **Latency > bandwidth** — adding bandwidth doesn't help if the round-trip is 200ms. Reduce requests.
- **Critical rendering path** — browser can't paint until CSS parsed and DOM ready. Minimize blocking resources.
- **HTTP/2 multiplexing** — one connection, many requests. Avoid domain sharding.

### Capacity (Nygard — Release It!)

- **Users ≠ sessions ≠ requests** — 10,000 users ≠ 10,000 concurrent sessions ≠ 10,000 req/s.
- **Response time vs throughput** — fast responses ≠ high throughput. Optimize for the constraint that matters.

## Inspired By

- **Designing Data-Intensive Applications** — Martin Kleppmann
- **High Performance Browser Networking** — Ilya Grigorik
- **Release It!** — Michael Nygard
- **Web Performance in Action** — Jeremy Wagner
- **High Performance MySQL** — Baron Schwartz et al.
```
