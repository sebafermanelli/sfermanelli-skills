---
name: sfermanelli-security-check
description: Audit code for security vulnerabilities and best practices. Use this skill when the user asks to check security, audit for vulnerabilities, review auth, check for XSS/SQL injection, review RLS policies, check for exposed secrets, or says things like "is this secure", "security audit", "check for vulnerabilities", "review permissions", "are there any security issues", "penetration test this code". Also triggers when reviewing auth flows, payment handling, or data access patterns.
---

# Security Check

Audit code for real security vulnerabilities — not theoretical concerns, but actual exploitable issues that could compromise data, money, or user trust.

## Golden Rule

**All user input is untrusted.** Even if it came from your own frontend, it could be manipulated. Validate at the boundary, sanitize before use, and never trust the client.

---

## 1. Audit Process

1. **Map the attack surface** — identify all entry points: API routes, server actions, form inputs, URL params, file uploads, webhooks
2. **Check each vulnerability class** — go through the checklist below systematically
3. **Assess severity** — not all issues are equal. A SQL injection is critical; a missing CSRF token on a read-only endpoint is low.
4. **Report with actionable fixes** — don't just say "this is vulnerable", show exactly how to fix it.

---

## 2. Vulnerability Checklist

### Authentication & Authorization

```markdown
- [ ] All protected routes check authentication
- [ ] User can only access their own data (not other users')
- [ ] Admin routes verify admin role/permissions
- [ ] OAuth callback validates state parameter (CSRF protection)
- [ ] Session tokens are httpOnly, secure, sameSite
- [ ] Password requirements enforced (min length, complexity)
- [ ] Rate limiting on login/signup endpoints
- [ ] Banned/suspended users cannot access the system
- [ ] Password reset tokens are single-use and expire
- [ ] MFA is available for sensitive operations
```

### IDOR (Insecure Direct Object Reference)

```typescript
// VULNERABLE — user can access any order by changing the ID
app.get('/api/orders/:id', async (req, res) => {
  const order = await prisma.order.findUnique({ where: { id: req.params.id } })
  return res.json(order) // Returns ANY order, regardless of who owns it
})

// SECURE — verify ownership
app.get('/api/orders/:id', async (req, res) => {
  const order = await prisma.order.findUnique({
    where: { id: req.params.id },
  })

  // Don't return 403 (reveals the order exists) — return 404
  if (!order || order.customerId !== req.user.id) {
    return res.status(404).json({ error: 'Order not found' })
  }

  return res.json(order)
})
```

### Data Access (RLS / Authorization)

```sql
-- Supabase RLS policy examples

-- VULNERABLE — allows anyone to see all orders
CREATE POLICY "Users can view orders" ON orders
FOR SELECT USING (true);

-- SECURE — users can only see their own orders
CREATE POLICY "Users can view own orders" ON orders
FOR SELECT USING (auth.uid() = customer_id);

-- SECURE — users can only insert their own orders
CREATE POLICY "Users can create own orders" ON orders
FOR INSERT WITH CHECK (auth.uid() = customer_id);

-- SECURE — users can only update their own orders, limited columns
CREATE POLICY "Users can update own orders" ON orders
FOR UPDATE USING (auth.uid() = customer_id)
WITH CHECK (auth.uid() = customer_id);

-- SECURE — store owners can manage their store's orders
CREATE POLICY "Store owners manage orders" ON orders
FOR ALL USING (
  EXISTS (
    SELECT 1 FROM stores
    WHERE stores.id = orders.store_id
    AND stores.owner_id = auth.uid()
  )
);
```

```markdown
RLS Checklist:

- [ ] Every table has RLS enabled
- [ ] SELECT policies don't leak data across users
- [ ] INSERT policies validate ownership (auth.uid() = user_id)
- [ ] UPDATE policies restrict which rows AND which columns
- [ ] DELETE policies exist and are restrictive
- [ ] Service client usage is justified and minimal
- [ ] No `security definer` functions without `search_path` set
- [ ] Junction tables have proper policies (not just parent tables)
```

### Input Validation

```typescript
// VULNERABLE — no validation
app.post('/api/orders', async (req, res) => {
  const order = await prisma.order.create({ data: req.body }) // Accepts ANYTHING
  return res.json(order)
})

// SECURE — validate with Zod at the boundary
const CreateOrderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        quantity: z.number().int().positive().max(100),
      })
    )
    .min(1)
    .max(50),
  shippingAddress: AddressSchema,
  notes: z.string().max(500).optional(),
})

app.post('/api/orders', async (req, res) => {
  const result = CreateOrderSchema.safeParse(req.body)
  if (!result.success) {
    return res.status(400).json({ error: formatZodError(result.error) })
  }
  const order = await orderService.create(result.data) // Validated data only
  return res.json(order)
})
```

```markdown
Validation Checklist:

- [ ] User inputs validated at system boundaries (server actions, API routes)
- [ ] File uploads: type checked, size limited, content validated (not just extension)
- [ ] URL parameters: parsed and validated, not used raw
- [ ] JSON payloads: schema validated (Zod, class-validator)
- [ ] String lengths limited (prevent memory exhaustion)
- [ ] Arrays have max length (prevent DoS)
- [ ] Numbers have min/max bounds
```

### Injection Attacks

#### XSS (Cross-Site Scripting)

```tsx
// VULNERABLE — renders user HTML directly
<div dangerouslySetInnerHTML={{ __html: userBio }} />

// VULNERABLE — javascript: URL scheme
<a href={userProvidedUrl}>Link</a>
// If userProvidedUrl = "javascript:alert('XSS')" → script executes

// SECURE — sanitize HTML if you must render it
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userBio) }} />

// SECURE — validate URLs
function isValidUrl(url: string): boolean {
  try {
    const parsed = new URL(url)
    return ['http:', 'https:'].includes(parsed.protocol)
  } catch {
    return false
  }
}
<a href={isValidUrl(url) ? url : '#'}>Link</a>

// React is safe by default — it escapes strings in JSX
// These are safe:
<p>{userInput}</p>          // Escaped automatically
<input value={userInput} /> // Escaped automatically
```

#### SQL Injection

```typescript
// VULNERABLE — string concatenation in query
const query = `SELECT * FROM users WHERE email = '${email}'`
// If email = "'; DROP TABLE users; --" → database deleted

// SECURE — parameterized queries (Prisma does this automatically)
const user = await prisma.user.findUnique({ where: { email } }) // Safe

// SECURE — parameterized raw query (when you need raw SQL)
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email = ${email}
` // Prisma parameterizes the template literal

// VULNERABLE — building raw query with string interpolation
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE name LIKE '%${search}%'` // NOT SAFE
)
// SECURE version:
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE name LIKE ${'%' + search + '%'}
`
```

#### Command Injection

```typescript
// VULNERABLE — user input in shell command
import { exec } from 'child_process'
exec(`convert ${userFilename} output.png`) // If filename = "; rm -rf /" → disaster

// SECURE — use execFile with array args (no shell interpretation)
import { execFile } from 'child_process'
execFile('convert', [userFilename, 'output.png']) // Arguments are NOT shell-interpreted

// SECURE — validate and sanitize filenames
const safeFilename = path.basename(userFilename) // Strips directory traversal
if (!/^[\w\-. ]+$/.test(safeFilename)) throw new Error('Invalid filename')
```

#### Path Traversal

```typescript
// VULNERABLE — user controls file path
app.get('/api/files/:name', (req, res) => {
  const filePath = path.join('/uploads', req.params.name)
  return res.sendFile(filePath)
})
// If name = "../../etc/passwd" → server files leaked

// SECURE — resolve and verify path is within allowed directory
app.get('/api/files/:name', (req, res) => {
  const uploadsDir = path.resolve('/uploads')
  const filePath = path.resolve(uploadsDir, req.params.name)

  // Verify the resolved path is still within uploads directory
  if (!filePath.startsWith(uploadsDir)) {
    return res.status(400).json({ error: 'Invalid file path' })
  }

  return res.sendFile(filePath)
})
```

### JWT Security

```typescript
// JWT Best Practices

// 1. Use strong signing algorithm
const token = jwt.sign(payload, secret, { algorithm: 'HS256' })  // Minimum
// Better: RS256 with asymmetric keys for service-to-service

// 2. Set short expiration
const token = jwt.sign(payload, secret, { expiresIn: '15m' })  // Access token
const refreshToken = jwt.sign(payload, secret, { expiresIn: '7d' })  // Refresh token

// 3. Always verify algorithm (prevent "none" algorithm attack)
const decoded = jwt.verify(token, secret, { algorithms: ['HS256'] })

// 4. Don't store sensitive data in JWT payload (it's base64, not encrypted)
// BAD:
{ userId: '123', creditCard: '4111...', ssn: '...' }
// GOOD:
{ userId: '123', role: 'admin', permissions: ['orders:read'] }

// 5. Store tokens securely
// Access token: in-memory (not localStorage — vulnerable to XSS)
// Refresh token: httpOnly, secure, sameSite cookie
res.cookie('refreshToken', token, {
  httpOnly: true,    // Not accessible via JavaScript
  secure: true,      // Only sent over HTTPS
  sameSite: 'strict', // Not sent in cross-site requests
  maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days
  path: '/api/auth/refresh'          // Only sent to refresh endpoint
})
```

### CORS Configuration

```typescript
// VULNERABLE — allows any origin
app.use(cors({ origin: '*' })) // Any website can make requests to your API

// VULNERABLE — reflects origin header (accepts any origin)
app.use(cors({ origin: true }))

// SECURE — explicit allowed origins
app.use(
  cors({
    origin: ['https://myapp.com', 'https://admin.myapp.com'],
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true, // Only if you need cookies/auth headers
    maxAge: 86400, // Cache preflight for 24 hours
  })
)

// NestJS CORS
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
  credentials: true,
})
```

### CSP (Content Security Policy)

```typescript
// Prevent XSS by controlling what resources can load
// In Next.js middleware or headers config:
const cspHeader = [
  "default-src 'self'",
  "script-src 'self' 'nonce-{NONCE}'", // Only allow scripts with nonce
  "style-src 'self' 'unsafe-inline'", // Inline styles (needed for many UI libs)
  "img-src 'self' blob: data: https:",
  "font-src 'self'",
  "connect-src 'self' https://api.stripe.com", // API endpoints
  "frame-ancestors 'none'", // Prevent clickjacking
  "base-uri 'self'",
  "form-action 'self'",
].join('; ')
```

### Secrets & Configuration

```markdown
- [ ] No hardcoded secrets in source code (grep for 'sk\_', 'key', 'secret', 'password')
- [ ] `.env` files in `.gitignore`
- [ ] API keys not exposed to client (`NEXT_PUBLIC_` only for truly public keys)
- [ ] Service role key / admin key never exposed to client
- [ ] JWT secrets are strong (>32 chars, random)
- [ ] Secrets rotated periodically
- [ ] Different secrets per environment (dev ≠ staging ≠ prod)
```

### Data Protection

```markdown
- [ ] Sensitive data (passwords, tokens, PII) not logged
- [ ] PII not stored unnecessarily
- [ ] File uploads stored in private buckets when appropriate
- [ ] Backup/export doesn't expose sensitive fields
- [ ] Passwords hashed with bcrypt/argon2 (NOT MD5/SHA)
- [ ] Deleted user data actually deleted (not just soft-deleted with full PII)
```

### Payment & Financial

```markdown
- [ ] Payment amounts calculated server-side, never from client
- [ ] Webhook signatures verified (Stripe, PayPal, etc.)
- [ ] Idempotency keys for payment operations
- [ ] Refund amounts validated against original payment
- [ ] No negative amounts accepted (could create credits)
- [ ] Price verified at checkout (not using price from cart/client)
```

### Dependency Security

```bash
# Check for known vulnerabilities
npm audit

# Fix automatically where possible
npm audit fix

# Check for outdated packages with known CVEs
npx better-npm-audit audit

# In CI/CD — fail build on high/critical vulnerabilities
npm audit --audit-level=high
```

### NestJS Security

```typescript
// Guards for authentication
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context)
  }
}

// Guards for authorization (role-based)
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler())
    if (!roles) return true

    const request = context.switchToHttp().getRequest()
    const user = request.user
    return roles.some(role => user.roles?.includes(role))
  }
}

// Usage
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  findAllUsers() { ... }
}

// Helmet for security headers
import helmet from 'helmet'
app.use(helmet())

// Rate limiting
import { ThrottlerModule } from '@nestjs/throttler'
@Module({
  imports: [
    ThrottlerModule.forRoot({ ttl: 60, limit: 100 })  // 100 requests per minute
  ]
})
```

### Angular Security

```typescript
// Angular auto-sanitizes by default — these are safe:
<p>{{ userInput }}</p>           <!-- Escaped -->
<div [innerHTML]="userHtml">    <!-- Sanitized by DomSanitizer -->

// VULNERABLE — bypassing sanitizer without good reason
constructor(private sanitizer: DomSanitizer) {}
this.sanitizer.bypassSecurityTrustHtml(userInput)  // DON'T do this with user input

// ONLY bypass when you control the content:
const trustedHtml = this.sanitizer.bypassSecurityTrustHtml(
  '<b>Server-generated content</b>'  // You control this, not the user
)

// HttpInterceptor for auth tokens
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken()
    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      })
    }
    return next.handle(req)
  }
}
```

---

## 3. Severity Levels

| Level        | Description                                       | Examples                                                                  | Response Time      |
| ------------ | ------------------------------------------------- | ------------------------------------------------------------------------- | ------------------ |
| **Critical** | Immediate exploitation possible, data breach risk | SQL injection, exposed secrets, missing auth on sensitive endpoints, RCE  | Fix immediately    |
| **High**     | Exploitation possible with some effort            | XSS with stored content, broken access control (IDOR), missing RLS        | Fix within 24h     |
| **Medium**   | Requires specific conditions to exploit           | CSRF on state-changing operations, information disclosure, weak passwords | Fix within 1 week  |
| **Low**      | Minor issues, defense in depth                    | Missing security headers, verbose error messages, unused permissions      | Fix in next sprint |

---

## 4. Report Format

````markdown
## Security Audit Report

### Critical Issues

🔴 **[CRITICAL] Missing authentication on `/api/admin/users`**

- **File:** `src/app/api/admin/users/route.ts:15`
- **OWASP:** A01 — Broken Access Control
- **Impact:** Any unauthenticated user can list all users and PII
- **Fix:**
  ```typescript
  const user = await getUser()
  if (!user?.isAdmin) return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  ```
````

### High Issues

🟠 **[HIGH] RLS policy allows users to see other users' orders**

- **Table:** `orders`
- **OWASP:** A01 — Broken Access Control
- **Policy:** `Users can view orders` uses `true` instead of `auth.uid() = customer_id`
- **Impact:** Any authenticated user can read all orders
- **Fix:** `USING (auth.uid() = customer_id)`

### Medium Issues

🟡 **[MEDIUM] File upload accepts any file type**

- **File:** `src/actions/upload.ts:35`
- **OWASP:** A04 — Insecure Design
- **Impact:** Users could upload executable files
- **Fix:** Validate MIME type server-side, not just extension

### Low Issues

🔵 **[LOW] Error message exposes database column names**

- **File:** `src/actions/orders.ts:142`
- **OWASP:** A05 — Security Misconfiguration
- **Fix:** Return generic error message to client

### No Issues Found

✅ Authentication flow — properly implemented
✅ Payment webhook — signature verified
✅ JWT — strong secret, short expiration, secure storage
✅ CORS — properly configured

```

---

## 5. OWASP Top 10 (2021) Reference

| # | Category | What to check |
|---|----------|--------------|
| A01 | **Broken Access Control** | RLS gaps, missing auth checks, IDOR, privilege escalation |
| A02 | **Cryptographic Failures** | Plaintext secrets, weak hashing, missing HTTPS, JWT in localStorage |
| A03 | **Injection** | SQL injection, XSS, command injection, path traversal |
| A04 | **Insecure Design** | Missing rate limiting, no abuse prevention, trust boundaries wrong |
| A05 | **Security Misconfiguration** | Default credentials, verbose errors, open CORS, missing headers |
| A06 | **Vulnerable Components** | Outdated deps with known CVEs (`npm audit`) |
| A07 | **Auth Failures** | Weak passwords, missing MFA, session fixation, token leaks |
| A08 | **Data Integrity Failures** | Unverified webhooks, unsigned updates, CI/CD tampering |
| A09 | **Logging Failures** | No audit trail, sensitive data in logs, no alerting |
| A10 | **SSRF** | Unvalidated URLs in server-side fetches, internal service access |

---

## 6. Principles

- **Focus on impact** — a theoretical vulnerability with no practical exploit path is low priority
- **Check the code, not the docs** — what the code actually does, not what it's supposed to do
- **Test the boundaries** — auth checks, RLS policies, input validation happen at system boundaries
- **Think like an attacker** — what would you try if you wanted to steal data or break the system?
- **Don't cry wolf** — reporting 50 "medium" issues dilutes the critical ones. Prioritize ruthlessly.
- **Defense in depth** — multiple layers of security. If one fails, others still protect.

### Secure by Design (Johnsson, Deogun, Sawano)

- **Domain primitives** — types that enforce constraints at creation. `Email` that validates is safer than a raw string.
- **Immutability as security** — data that can't be modified can't be tampered with.
- **Fail securely** — when something goes wrong, deny access. A crashed auth check should block, not allow.

## Inspired By

- **OWASP Top 10** (2021)
- **The Web Application Hacker's Handbook** — Dafydd Stuttard & Marcus Pinto
- **Secure by Design** — Dan Bergh Johnsson, Daniel Deogun & Daniel Sawano
- **OWASP Cheat Sheet Series** — cheatsheetseries.owasp.org
- **Web Security for Developers** — Malcolm McDonald
```
