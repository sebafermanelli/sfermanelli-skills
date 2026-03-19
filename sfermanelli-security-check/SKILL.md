---
name: sfermanelli-security-check
description: Audit code for security vulnerabilities and best practices. Use this skill when the user asks to check security, audit for vulnerabilities, review auth, check for XSS/SQL injection, review RLS policies, check for exposed secrets, or says things like "is this secure", "security audit", "check for vulnerabilities", "review permissions", "are there any security issues", "penetration test this code". Also triggers when reviewing auth flows, payment handling, or data access patterns.
---

# Security Check

Audit code for real security vulnerabilities — not theoretical concerns, but actual exploitable issues that could compromise data, money, or user trust.

## Audit Process

1. **Map the attack surface** — identify all entry points: API routes, server actions, form inputs, URL params, file uploads, webhooks
2. **Check each vulnerability class** — go through the checklist below systematically
3. **Assess severity** — not all issues are equal. A SQL injection is critical; a missing CSRF token on a read-only endpoint is low.
4. **Report with actionable fixes** — don't just say "this is vulnerable", show exactly how to fix it.

## Vulnerability Checklist

### Authentication & Authorization
- [ ] All protected routes check authentication
- [ ] User can only access their own data (not other users')
- [ ] Admin routes verify `is_platform_admin`
- [ ] OAuth callback validates state parameter
- [ ] Session tokens are httpOnly, secure, sameSite
- [ ] Password requirements enforced (min length)
- [ ] Rate limiting on login/signup endpoints
- [ ] Banned users cannot access the system

### Data Access (RLS / Authorization)
- [ ] Every table has RLS enabled
- [ ] SELECT policies don't leak data across users
- [ ] INSERT policies validate ownership (`auth.uid() = user_id`)
- [ ] UPDATE policies restrict which rows AND which columns
- [ ] DELETE policies exist and are restrictive
- [ ] Service client usage is justified and minimal
- [ ] No `security definer` functions without `search_path` set

### Input Validation
- [ ] User inputs validated at system boundaries (server actions, API routes)
- [ ] File uploads: type checked, size limited, content validated (not just extension)
- [ ] URL parameters: parsed and validated, not used raw
- [ ] JSON payloads: schema validated
- [ ] SQL: parameterized queries only (Supabase handles this, but check raw queries)
- [ ] HTML: no `dangerouslySetInnerHTML` with user content

### Injection Attacks
- [ ] **XSS**: User content rendered safely (React does this by default, but check `dangerouslySetInnerHTML`, `href="javascript:..."`)
- [ ] **SQL Injection**: All queries use parameterized inputs (check `.rpc()` calls, raw SQL)
- [ ] **Command Injection**: No `exec()`, `spawn()` with user input
- [ ] **Path Traversal**: File paths validated, no `../` in user-provided paths

### Secrets & Configuration
- [ ] No hardcoded secrets in source code
- [ ] `.env` files in `.gitignore`
- [ ] API keys not exposed to client (`NEXT_PUBLIC_` only for truly public keys)
- [ ] Service role key never exposed to client
- [ ] JWT secrets are strong (>32 chars)

### Data Protection
- [ ] Sensitive data (passwords, tokens) not logged
- [ ] PII not stored unnecessarily
- [ ] File uploads stored in private buckets when appropriate
- [ ] Backup/export doesn't expose sensitive fields

### Payment & Financial
- [ ] Payment amounts calculated server-side, never from client
- [ ] Webhook signatures verified
- [ ] Idempotency keys for payment operations
- [ ] Refund amounts validated against original payment

### Infrastructure
- [ ] CORS configured correctly (not `*` in production)
- [ ] Content Security Policy headers
- [ ] HTTPS enforced
- [ ] Error messages don't leak internal details to users

## Severity Levels

| Level | Description | Examples |
|-------|------------|---------|
| **Critical** | Immediate exploitation possible, data breach risk | SQL injection, exposed secrets, missing auth on sensitive endpoints |
| **High** | Exploitation possible with some effort | XSS with stored content, broken access control, missing RLS |
| **Medium** | Requires specific conditions to exploit | CSRF on state-changing operations, information disclosure |
| **Low** | Minor issues, defense in depth | Missing security headers, verbose error messages |

## Report Format

```markdown
## Security Audit Report

### Critical Issues
🔴 **[CRITICAL] Missing authentication on `/api/admin/users`**
- **File:** `src/app/api/admin/users/route.ts:15`
- **Impact:** Any unauthenticated user can list all users
- **Fix:**
  ```typescript
  const user = await getUser()
  if (!user?.isAdmin) return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  ```

### High Issues
🟠 **[HIGH] RLS policy allows users to see other users' orders**
- **Table:** `orders`
- **Policy:** `Users can view orders` uses `true` instead of `auth.uid() = client_id`
- **Fix:** Update the policy USING clause

### Medium Issues
🟡 **[MEDIUM] File upload accepts any file type**
- **File:** `src/actions/contact.ts:35`
- **Impact:** Users could upload executable files
- **Fix:** Validate MIME type, not just extension

### Low Issues
🔵 **[LOW] Error message exposes database column names**
- **File:** `src/actions/orders.ts:142`
- **Fix:** Return generic error message to client

### No Issues Found
✅ Authentication flow — properly implemented
✅ Payment webhook — signature verified
✅ Environment variables — properly configured
```

## Principles

- **Focus on impact** — a theoretical vulnerability with no practical exploit path is low priority
- **Check the code, not the docs** — what the code actually does, not what it's supposed to do
- **Test the boundaries** — auth checks, RLS policies, input validation happen at system boundaries
- **Think like an attacker** — what would you try if you wanted to steal data or break the system?
- **Don't cry wolf** — reporting 50 "medium" issues dilutes the critical ones. Prioritize ruthlessly.

## OWASP Top 10 (2021) Reference

Map findings to the official OWASP categories for industry-standard reporting:

| # | Category | What to check |
|---|----------|--------------|
| A01 | **Broken Access Control** | RLS gaps, missing auth checks, IDOR, privilege escalation |
| A02 | **Cryptographic Failures** | Plaintext secrets, weak hashing, missing HTTPS |
| A03 | **Injection** | SQL injection, XSS, command injection, LDAP injection |
| A04 | **Insecure Design** | Missing rate limiting, no abuse prevention, trust boundaries |
| A05 | **Security Misconfiguration** | Default credentials, verbose errors, open CORS |
| A06 | **Vulnerable Components** | Outdated deps with known CVEs (`npm audit`) |
| A07 | **Auth Failures** | Weak passwords, missing MFA, session fixation |
| A08 | **Data Integrity Failures** | Unverified webhooks, unsigned updates |
| A09 | **Logging Failures** | No audit trail, sensitive data in logs |
| A10 | **SSRF** | Unvalidated URLs in server-side fetches |

## Secure by Design Principles

From **Secure by Design** (Johnsson, Deogun, Sawano):

- **Domain primitives** — use types that enforce constraints at creation. An `EmailAddress` type that validates on construction is safer than a raw string that might be validated somewhere.
- **Immutability as security** — data that can't be modified can't be tampered with.
- **Fail securely** — when something goes wrong, deny access rather than grant it. A crashed auth check should block, not allow.
- **"All user input is untrusted"** (The Web Application Hacker's Handbook) — even if it came from your own frontend, it could be manipulated.

## Inspired By

- **OWASP Top 10** (2021)
- **The Web Application Hacker's Handbook** — Dafydd Stuttard & Marcus Pinto
- **Secure by Design** — Dan Bergh Johnsson, Daniel Deogun & Daniel Sawano
