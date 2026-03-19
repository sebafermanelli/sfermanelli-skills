---
name: sfermanelli-performance
description: Identify and fix performance bottlenecks in code, queries, and rendering. Use this skill when the user asks to optimize performance, speed up the app, fix slow queries, reduce bundle size, improve load time, or says things like "this is slow", "why is it taking so long", "optimize this", "the page is laggy", "reduce loading time", "improve speed", "find bottlenecks". Also triggers for N+1 queries, unnecessary re-renders, large bundles, or database optimization.
---

# Performance

Find and fix the bottlenecks that actually matter — not micro-optimizations that save nanoseconds, but changes that users can feel.

## Rule of Thumb

**Measure before optimizing.** If you can't measure the improvement, you can't prove the optimization worked. Premature optimization is the root of all evil — but ignoring a 3-second page load is negligence.

## Performance Audit Process

1. **Identify the symptom** — what's slow? Page load? API response? UI interaction?
2. **Measure the baseline** — how slow is it? Consistently or intermittently?
3. **Find the bottleneck** — profile, don't guess. The bottleneck is rarely where you think.
4. **Fix the bottleneck** — apply the minimum change for maximum impact.
5. **Measure the improvement** — did it get faster? By how much?

## Common Bottlenecks by Layer

### Database / Queries

**N+1 Queries** — the #1 backend performance killer:
```typescript
// BAD: 1 query for stores + N queries for reviews
const stores = await supabase.from('stores').select('*')
for (const store of stores) {
  const reviews = await supabase.from('reviews').select('*').eq('store_id', store.id)
}

// GOOD: 1 query with join
const stores = await supabase.from('stores').select('*, reviews(rating)')
```

**Missing indexes:**
```sql
-- If you filter or join on a column frequently, index it
CREATE INDEX idx_orders_store_id ON orders(store_id);
CREATE INDEX idx_orders_status ON orders(status) WHERE status IN ('pending', 'confirmed');
```

**Over-fetching:**
```typescript
// BAD: fetching everything
const { data } = await supabase.from('orders').select('*')

// GOOD: fetch only what you need
const { data } = await supabase.from('orders').select('id, status, total_charged, created_at')
```

**Sequential queries that could be parallel:**
```typescript
// BAD: sequential
const stores = await getStores()
const orders = await getOrders()
const users = await getUsers()

// GOOD: parallel
const [stores, orders, users] = await Promise.all([
  getStores(),
  getOrders(),
  getUsers(),
])
```

### React / Frontend

**Unnecessary re-renders:**
```typescript
// BAD: creates new object every render, children re-render
<Context.Provider value={{ user, setUser }}>

// GOOD: memoize the value
const value = useMemo(() => ({ user, setUser }), [user])
<Context.Provider value={value}>
```

**Large component trees:**
- Split heavy components with `React.lazy()` and `Suspense`
- Use `loading.tsx` in Next.js for route-level code splitting
- Move expensive computations to `useMemo`

**Bundle size:**
```bash
# Analyze what's in your bundle
npx next build && npx @next/bundle-analyzer
```
- Dynamic import heavy libraries: `const Leaflet = dynamic(() => import('leaflet'), { ssr: false })`
- Check for duplicate dependencies: `npx depcheck`
- Tree-shake: import `{ specific }` not `import *`

### Next.js Specific

**Server vs Client components:**
- Data fetching → server component (no client JS shipped)
- Interactivity → client component (minimal)
- Don't make a whole page client-side for one interactive element — split it

**Caching:**
```typescript
// Static data: cache aggressively
export const revalidate = 3600 // revalidate every hour

// Dynamic data: use ISR or on-demand revalidation
revalidatePath('/stores')
```

**Image optimization:**
```typescript
// BAD
<img src={url} />

// GOOD
<Image src={url} width={400} height={300} alt="..." />
// Next.js auto-optimizes: WebP, lazy loading, responsive sizes
```

### API / Network

**Payload size:**
- Return only the fields the client needs
- Paginate lists (don't return 10,000 records)
- Compress responses (usually handled by the framework)

**Waterfall requests:**
- Fetch data in parallel on the server, not sequentially on the client
- Use server components for data-heavy pages
- Prefetch links with `<Link prefetch>`

## Performance Report

```markdown
## Performance Audit

### Bottleneck Found
**Symptom:** Store detail page takes 3.2s to load
**Root cause:** Sequential queries — getStoreInfo, getProducts, getReviews run one after another
**Impact:** ~2s wasted waiting

### Fix Applied
Parallelized with `Promise.all`:
```typescript
const [store, products, reviews] = await Promise.all([
  getStoreInfo(slug),
  getProducts(storeId),
  getReviews(storeId),
])
```

### Result
- Before: 3.2s
- After: 1.1s
- Improvement: 66% faster

### Additional Recommendations
1. Add index on `products.store_id` (currently missing)
2. Consider caching store info with `revalidate = 60`
3. Lazy load review section below the fold
```

## Principles

- **Optimize for the user, not the machine** — 100ms improvement on a 200ms operation is invisible. 1s improvement on a 4s operation is transformative.
- **The fastest code is code that doesn't run** — can you cache it? Can you avoid the computation entirely?
- **Measure in production conditions** — dev mode is not representative. SQLite is not Postgres.
- **Network is usually the bottleneck** — reduce round trips before optimizing CPU.
- **Don't sacrifice readability for speed** — unless the performance gain is significant and measured.

## Data Layer Patterns (Martin Kleppmann — Designing Data-Intensive Applications)

### Indexing Strategy
- **B-tree indexes** (default) — great for equality and range queries
- **Partial indexes** — index only the rows you query: `WHERE status = 'active'`
- **Composite indexes** — column order matters: put equality columns first, range columns last
- **Cover your queries** — if a query only needs indexed columns, it never touches the table

### Caching Patterns
- **Cache-aside** — app checks cache first, loads from DB on miss, writes to cache
- **Write-through** — write to cache and DB together, read from cache
- **TTL-based** — set expiration, accept stale data within the window

### Denormalization Tradeoffs
Normalize for writes, denormalize for reads. If a query joins 5 tables and runs 1000x/day, consider storing the precomputed result.

## Network Performance (Ilya Grigorik — High Performance Browser Networking)

- **Latency > bandwidth** — adding bandwidth doesn't help if the round-trip time is 200ms. Reduce the number of requests.
- **Critical rendering path** — the browser can't paint until CSS is parsed and the DOM is ready. Minimize render-blocking resources.
- **Connection reuse** — HTTP/2 multiplexes requests over one connection. Avoid domain sharding.

## Capacity Thinking (Michael Nygard — Release It!)

- **Users ≠ sessions ≠ requests** — 10,000 users ≠ 10,000 concurrent sessions ≠ 10,000 requests/second. Understand the conversion ratios.
- **Response time vs throughput** — you can have fast responses with low throughput (few users) or slow responses with high throughput (many users). Optimize for the constraint that matters.

## Inspired By

- **Designing Data-Intensive Applications** — Martin Kleppmann
- **High Performance Browser Networking** — Ilya Grigorik
- **Release It!** — Michael Nygard
- **Web Performance in Action** — Jeremy Wagner
