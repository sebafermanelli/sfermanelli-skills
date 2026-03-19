---
name: sfermanelli-review-pr
description: Review pull requests and code changes like a senior engineer. Use this skill when the user asks to review a PR, review code changes, check a diff, look at what changed, review before merging, or anything involving code review. Also triggers for "review this", "check my changes", "is this ready to merge", "what do you think of this diff", or "give me feedback on this code".
---

# Review PR

Review code changes with the rigor of a senior engineer who cares about the codebase long-term — not a nitpicker who blocks PRs over style preferences.

> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." — Martin Fowler

## Golden Rule

**Review the change, not the person.** Focus on the code and its impact. Every comment should help the author make the code better, not make them feel worse.

---

## 1. Review Process

1. **Understand the context** — read the PR description, linked issues, and commit messages. What problem is being solved? Why this approach?
2. **Read the diff top-down** — understand the overall change before diving into details.
3. **Check against the codebase** — how does this change fit into the broader architecture? Does it follow existing patterns?
4. **Run the checks** — `npm run lint`, `npx tsc --noEmit`, `npm run build`, `npm test` if appropriate.
5. **Review the tests** — are the right things tested? Are edge cases covered?
6. **Deliver the review** — structured, actionable, respectful.

---

## 2. What to Look For

### Critical (Must fix before merge)

- **Bugs** — logic errors, off-by-one, race conditions, unhandled nulls
- **Security** — SQL injection, XSS, exposed secrets, missing auth checks, RLS gaps, IDOR
- **Data loss** — destructive operations without confirmation, missing cascading deletes, irreversible mutations
- **Breaking changes** — public API changes without versioning, DB schema changes without migration
- **Correctness** — does the code actually do what the PR description says?

### Important (Should fix)

- **Missing error handling** — what happens when the API call fails? When the record doesn't exist?
- **Missing validation** — user inputs not validated at system boundaries
- **Performance** — N+1 queries, unnecessary re-renders, missing indexes, over-fetching
- **Incomplete implementation** — TODOs left in code, feature flags not wired up, missing edge cases
- **Test gaps** — critical paths untested, only happy path covered, mocking too much

### Suggestions (Nice to have)

- **Readability** — confusing names, overly clever code, missing comments on non-obvious logic
- **Consistency** — deviating from project patterns without good reason
- **Simplification** — code that could be simpler without losing clarity
- **Better abstractions** — opportunities to extract shared logic

### Don't Review

- Style preferences already enforced by linter/formatter (let tooling handle it)
- Trailing whitespace, import ordering, semicolons
- "I would have done it differently" without a concrete benefit
- Unrelated code that wasn't changed in this PR (file a separate issue)

---

## 3. Review Checklist by Change Type

### API / Backend Changes

```markdown
- [ ] Endpoint follows REST conventions (naming, methods, status codes)
- [ ] Request validation present (Zod, class-validator, etc.)
- [ ] Authentication and authorization checked
- [ ] Error responses are consistent with other endpoints
- [ ] Database queries are efficient (no N+1, proper indexes)
- [ ] Transactions used where multiple writes must be atomic
- [ ] Idempotency handled for create/payment operations
- [ ] Rate limiting considered for public endpoints
- [ ] API versioning if breaking change
```

### UI / Frontend Changes

```markdown
- [ ] Component handles loading, error, and empty states
- [ ] Accessible: labels, keyboard navigation, screen reader support
- [ ] Responsive: works on mobile, tablet, desktop
- [ ] No unnecessary re-renders (check React DevTools Profiler)
- [ ] User-facing strings are translatable (if i18n applies)
- [ ] Forms validate on submit AND on blur for critical fields
- [ ] Optimistic updates revert on error
- [ ] Images use next/image or equivalent optimization
```

### Database Migration Changes

```markdown
- [ ] Migration is reversible (can rollback without data loss)
- [ ] No breaking changes to running code (expand-contract pattern)
- [ ] New columns are nullable OR have defaults
- [ ] Indexes added for new query patterns
- [ ] Large table migrations done in batches (not one ALTER on 10M rows)
- [ ] RLS policies updated if new tables/columns added
- [ ] Backfill script included if needed
- [ ] Tested against production-size data (not just empty DB)
```

### Dependency Updates

```markdown
- [ ] Changelog/migration guide read for breaking changes
- [ ] Lock file updated (package-lock.json, yarn.lock)
- [ ] No security vulnerabilities in new version (npm audit)
- [ ] Type changes checked (especially for major versions)
- [ ] Tests pass with new version
- [ ] Bundle size impact checked (especially for frontend deps)
```

### Test Changes

```markdown
- [ ] Tests test behavior, not implementation details
- [ ] Test names describe WHAT should happen, not HOW
- [ ] Edge cases covered (null, empty, boundary values)
- [ ] Error paths tested, not just happy path
- [ ] Mocks are minimal — only mock external services
- [ ] No flaky patterns (timeouts, race conditions, global state)
- [ ] Tests can run in any order and in parallel
```

---

## 4. Conventional Comments

Use prefixes to communicate intent clearly. The author immediately knows what needs action vs what's optional.

````markdown
**issue:** This will throw if `user` is null. `getUser()` returns null when session expires.

```suggestion
const user = await getUser()
if (!user) return unauthorized()
```
````

**suggestion:** Consider using `Promise.allSettled` here so one failure doesn't block the others.

**question:** Is there a reason this doesn't use the existing `OrderMapper`? Wondering if I'm missing context.

**nitpick:** Typo in variable name: `recieve` → `receive`. (Non-blocking, merge anyway.)

**praise:** Great error handling here — the fallback to cached data is exactly the right approach.

**thought:** This pattern is going to get complex as we add more payment providers. Might be worth extracting a strategy pattern at some point. (Not for this PR.)

**todo:** We'll need to add an index on `orders.customer_id` before this goes to production with real traffic. Can be a follow-up PR.

````

### Comment Prefix Reference

| Prefix | Meaning | Blocking? |
|--------|---------|-----------|
| `issue:` | Bug, security problem, or correctness issue | Yes — must fix |
| `suggestion:` | Alternative approach that's better but not critical | No — author decides |
| `question:` | Seeking understanding, might be an issue or might be fine | No — but needs answer |
| `nitpick:` | Style, minor improvement, bikeshedding | No — merge anyway |
| `praise:` | Something done well | No — never blocking |
| `thought:` | Future consideration, not for this PR | No — FYI only |
| `todo:` | Follow-up work needed (can be merged now) | No — file a ticket |

---

## 5. Review Examples

### Good Review Comments

```markdown
# Specific, actionable, shows the impact
**issue:** `src/services/order.service.ts:45`
This query returns all orders without pagination. With 100K+ orders in production,
this will timeout and crash the endpoint.
```suggestion
const orders = await prisma.order.findMany({
  where: { storeId },
  take: params.pageSize,
  skip: (params.page - 1) * params.pageSize,
  orderBy: { createdAt: 'desc' }
})
````

# Context-aware suggestion

**suggestion:** `src/components/OrderList.tsx:23`
Since this data comes from a server component, you don't need `useEffect` + `useState`
for fetching. You can pass it directly as props and remove 15 lines of loading state management.

# Good question — not a demand

**question:** `src/middleware.ts:12`
This redirects unauthenticated users to /login, but what about the /api/webhooks
endpoint? Stripe needs to hit that without auth. Should we exclude it from the matcher?

````

### Bad Review Comments

```markdown
# Too vague — author doesn't know what to fix
"This doesn't look right."
"I'm not sure about this approach."

# Demanding without reason
"Use reduce instead of forEach."  ← Why? forEach is more readable here.

# Reviewing style when there's a linter
"Add a semicolon here."  ← That's the formatter's job.

# Scope creep — reviewing unrelated code
"While you're here, can you also refactor the UserService?"  ← File a separate issue.

# Personal preference without benefit
"I prefer interfaces over types."  ← Unless there's a concrete reason, this is noise.
````

---

## 6. PR Size and Structure

### Size Guidelines

| Lines Changed | Assessment | Action                                                     |
| ------------- | ---------- | ---------------------------------------------------------- |
| 1-50          | Small      | Review immediately, easy to understand                     |
| 50-200        | Medium     | Ideal PR size. Full review.                                |
| 200-500       | Large      | Careful review. Consider if it should be split.            |
| 500+          | Too large  | Ask author to split. Review quality degrades at this size. |

### When to Ask for a Split

- PR mixes unrelated changes (feature + refactor + dependency update)
- PR touches too many different areas of the codebase
- Migration and code changes in the same PR
- Hard to understand what changed and why

```markdown
# Asking for a split — be helpful, not demanding

"This PR has a lot going on — the migration, the new endpoint, and the refactor of
OrderService. Would it be possible to split into 3 PRs? That way we can merge the
migration first and verify it works before the code changes land. Happy to review
each one quickly."
```

### What Makes a Good PR (for the author)

- **Clear title** — under 70 characters, describes the change
- **Description** — what changed, why, how to test
- **Small and focused** — one logical change per PR
- **Tests included** — or explanation of why not
- **Self-reviewed** — author reviewed their own diff before requesting review
- **No unrelated changes** — formatting, refactoring, or cleanups in separate PRs

---

## 7. Review Output Format

````markdown
## PR Review: [brief description of what the PR does]

### Summary

[2-3 sentences: what the PR does, overall assessment, recommendation]

**Recommendation:** Approve / Request changes / Approve with suggestions

### Critical (must fix)

- **issue:** `src/services/order.service.ts:45` — Query returns all orders without
  pagination. Will timeout on production data.
  ```suggestion
  // suggested fix
  ```
````

### Important (should fix)

- **issue:** `src/controllers/order.controller.ts:23` — Missing error handling for
  `findById` returning null.
- **suggestion:** `src/services/payment.service.ts:67` — Consider adding idempotency
  key for the charge operation.

### Suggestions (non-blocking)

- **nitpick:** `src/utils/format.ts:12` — `recieve` → `receive`
- **thought:** The OrderMapper might benefit from a `toSummaryDto()` method as the
  list endpoint grows.

### Tests

- **todo:** The error path (order not found) isn't tested. Worth adding.
- **praise:** Good coverage of edge cases in the pricing tests.

### What's Good

- Clean separation between validation and business logic
- Good use of the existing `Money` value object
- Error messages are user-friendly and consistent

```

---

## 8. Principles

### Review Philosophy

- **Be specific** — "this might have issues" is useless. "This will throw if `user` is null on line 45 because `getUser()` returns null when the session expires" is useful.
- **Suggest, don't dictate** — show the fix, explain why, let the author decide.
- **Praise good work** — if the error handling is thorough or the naming is excellent, say so. Good PR reviews acknowledge quality.
- **Context matters** — a prototype doesn't need the same rigor as a payment flow.
- **One round** — aim to catch everything in one pass. Back-and-forth reviews waste time.
- **Be timely** — review within hours, not days. A PR waiting for review is blocked work.

### From the Masters

- **Broken windows** (The Pragmatic Programmer) — flag code that will deteriorate. One hack invites another.
- **Deep vs shallow modules** (Ousterhout) — a good module has a simple interface hiding complex implementation. If a PR adds complexity to the interface without adding capability, push back.
- **Tactical vs strategic** (Ousterhout) — is this PR a quick hack that adds tech debt, or a well-thought investment? Both have their place, but name it.
- **Boy Scout Rule** (Clean Code) — leave the code better than you found it. But in a review context: don't demand perfection, ask for improvement.
- **The code is the design** (Jack Reeves) — the code IS the product. Reviewing code is reviewing the product itself.

### Reviewing for Growth

When reviewing code from junior developers:

- **Teach through reviews** — explain WHY something is a problem, not just that it is. Link to relevant patterns or docs.
- **Distinguish blocking from educational** — "This will crash in production" vs "Here's a pattern that might serve you well in the future."
- **One learning point per review** — don't overwhelm with 20 educational comments. Pick the most valuable lesson.

---

## Inspired By

- **Code Complete** — Steve McConnell (Peer Reviews)
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **A Philosophy of Software Design** — John Ousterhout
- **Refactoring** — Martin Fowler
- **Thoughtbot Code Review Guide** — thoughtbot.com/blog/code-review
- **Conventional Comments** — conventionalcomments.org
- **Google Engineering Practices** — Code Review Developer Guide
```
