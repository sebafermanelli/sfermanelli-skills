---
name: sfermanelli-debug
description: Investigate and fix bugs by tracing root causes, not just symptoms. Use this skill when the user reports a bug, error, unexpected behavior, or something that "doesn't work". Triggers for "this is broken", "why isn't this working", "I'm getting an error", "it crashes when", "unexpected behavior", "this used to work", "can you debug this", stack traces, error screenshots, or any problem-solving scenario. Different from fix-lint (which is automated) — this is investigation and diagnosis.
---

# Debug

Find and fix bugs by understanding the root cause — not by guessing, not by adding try-catch everywhere, and definitely not by rewriting the whole thing.

## Investigation Process

### 1. Reproduce
Before fixing anything, understand the problem:
- What's the expected behavior?
- What's the actual behavior?
- What are the steps to reproduce?
- Does it happen consistently or intermittently?

### 2. Isolate
Narrow down where the bug lives:
- **Read the error message** — the answer is often right there. Stack traces, line numbers, error codes.
- **Trace the data flow** — follow the data from input to output. Where does it diverge from expected?
- **Check the boundaries** — bugs cluster at boundaries: API ↔ client, server ↔ DB, component ↔ state
- **Check recent changes** — `git log --oneline -20`, `git diff HEAD~5`. What changed?

### 3. Diagnose
Identify the root cause, not just the symptom:
- **Symptom:** "the page shows a 500 error"
- **Surface cause:** "the API route throws"
- **Root cause:** "the query uses `renter_id` but the column was renamed to `client_id` in migration 00038"

Ask "why" until you can't anymore. The fix for the root cause is always better than a fix for the symptom.

### 4. Fix
Apply the minimal change that fixes the root cause:
- Don't refactor while debugging — fix the bug, then clean up in a separate step
- Don't add defensive code that hides the real problem
- If the fix is more than ~20 lines, reconsider — you might be fixing the wrong thing

### 5. Verify
- Confirm the fix resolves the original issue
- Check for regressions — did fixing this break something else?
- Consider edge cases — does the fix work for all inputs, not just the reported one?

## Common Bug Patterns

### Async/Timing
- `await` missing → promise returned instead of value
- State read before hydration (SSR vs client mismatch)
- Race condition between parallel operations
- Stale closure capturing old state value

### Type Coercion
- `==` instead of `===`
- String "0" being truthy, `null` vs `undefined` behavior
- `parseInt("08")` without radix
- Date objects vs ISO strings vs timestamps

### Database
- Column renamed but code not updated
- RLS policy blocking access (works with service key, fails with user key)
- Missing index causing timeout
- FK constraint preventing delete

### React
- Missing `key` prop causing stale renders
- `useEffect` missing dependency → stale data
- Server component using client-only API (hooks, window, document)
- Hydration mismatch (server renders X, client renders Y)

### Next.js
- `headers()` / `cookies()` not awaited (Next.js 15+)
- Dynamic import needed for client-only libraries (Leaflet, etc.)
- Middleware running on routes it shouldn't
- `notFound()` interaction with error boundaries

## Debug Output

```markdown
## Bug Report

**Symptom:** [what the user sees]
**Root cause:** [the actual problem]
**Fix:** [what was changed and why]

### Investigation trace:
1. [first thing checked and what was found]
2. [second thing checked]
3. [the "aha" moment — where the root cause was identified]

### Files changed:
- `path/to/file.ts:42` — [what was changed]

### Verified:
- ✓ Original issue resolved
- ✓ No regressions detected
- ✓ Edge cases considered
```

## Principles

- **Read before writing** — understand the code before changing it
- **One fix at a time** — don't bundle unrelated changes with the bug fix
- **Log strategically** — add temporary logs to trace data flow, remove them after
- **Don't guess** — if you're not sure, add a log and reproduce. Guessing wastes time.
- **The simplest explanation is usually right** — before suspecting a framework bug, check your code
- **"select() isn't broken"** (The Pragmatic Programmer) — it's almost certainly YOUR code, not the framework, OS, or compiler
- **"Don't assume it, prove it"** (The Pragmatic Programmer) — verify every assumption with evidence
- **Rubber duck debugging** (The Pragmatic Programmer) — explain the problem out loud, step by step. The act of explaining often reveals the answer.
- **Binary search for bugs** — use `git bisect` to find the exact commit that introduced the bug

### Agans' 9 Rules of Debugging
From **Debugging** by David Agans:
1. **Understand the system** — read the docs, know the architecture
2. **Make it fail** — find a reliable reproduction
3. **Quit thinking and look** — observe actual behavior, don't theorize
4. **Divide and conquer** — narrow down systematically
5. **Change one thing at a time** — scientific method
6. **Keep an audit trail** — write down what you tried and what happened
7. **Check the plug** — verify the obvious first (is the server running? is the env var set?)
8. **Get a fresh view** — ask someone else, or step away and come back
9. **If you didn't fix it, it ain't fixed** — verify the fix, don't assume

### Stability Patterns (Release It! — Michael Nygard)
When debugging production issues, understand failure cascades:
- **Bulkheads** — isolate failures so one broken service doesn't take everything down
- **Circuit breakers** — stop calling a failing service, fail fast instead
- **Timeouts** — always set timeouts on external calls. No timeout = infinite hang risk.

## Inspired By

- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **Debugging** — David Agans
- **Release It!** — Michael Nygard
