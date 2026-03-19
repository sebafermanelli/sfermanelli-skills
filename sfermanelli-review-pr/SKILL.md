---
name: sfermanelli-review-pr
description: Review pull requests and code changes like a senior engineer. Use this skill when the user asks to review a PR, review code changes, check a diff, look at what changed, review before merging, or anything involving code review. Also triggers for "review this", "check my changes", "is this ready to merge", "what do you think of this diff", or "give me feedback on this code".
---

# Review PR

Review code changes with the rigor of a senior engineer who cares about the codebase long-term — not a nitpicker who blocks PRs over style preferences.

## Review Process

1. **Understand the context** — read the PR description, linked issues, and commit messages. What problem is being solved? Why this approach?
2. **Read the diff** — `git diff main...HEAD` or `git diff --staged`. Understand every change.
3. **Check the existing codebase** — how does this change fit into the broader architecture? Does it follow existing patterns?
4. **Run the checks** — `npm run lint`, `npx tsc --noEmit`, `npm run build` if appropriate.
5. **Deliver the review** — structured, actionable, respectful.

## What to Look For

### Critical (Must fix before merge)
- **Bugs** — logic errors, off-by-one, race conditions, unhandled nulls
- **Security** — SQL injection, XSS, exposed secrets, missing auth checks, RLS gaps
- **Data loss** — destructive operations without confirmation, missing cascading deletes
- **Breaking changes** — public API changes, DB schema changes without migration

### Important (Should fix)
- **Missing error handling** — what happens when the API call fails? When the file doesn't exist?
- **Missing validation** — user inputs not validated at system boundaries
- **Performance** — N+1 queries, unnecessary re-renders, missing indexes
- **Incomplete implementation** — TODOs left in code, feature flags not wired up

### Suggestions (Nice to have)
- **Readability** — confusing variable names, overly clever code, missing comments on non-obvious logic
- **Consistency** — deviating from project patterns without good reason
- **Simplification** — code that could be simpler without losing clarity

### Explicitly Don't Review
- Style preferences already enforced by linter/formatter
- Trailing whitespace, import ordering (let tooling handle it)
- "I would have done it differently" without a concrete reason

## Review Output Format

```markdown
## PR Review: [brief description of what the PR does]

### Summary
[2-3 sentences: what the PR does, overall assessment]

### Critical
- **[file:line]** — [description of the issue and why it matters]
  ```suggestion
  // suggested fix
  ```

### Important
- **[file:line]** — [description and suggestion]

### Suggestions
- **[file:line]** — [description]

### What's Good
- [call out something well-done — good PR reviews acknowledge quality]
```

## Principles

- **Be specific** — "this might have issues" is useless. "This will throw if `user` is null on line 45 because `getUser()` returns null when the session expires" is useful.
- **Suggest, don't dictate** — show the fix, explain why, let the author decide.
- **Praise good work** — if the error handling is thorough or the naming is excellent, say so.
- **Context matters** — a prototype doesn't need the same rigor as a payment flow.
- **One round** — aim to catch everything in one pass. Back-and-forth reviews waste time.
- **Broken windows** (The Pragmatic Programmer) — flag code that will deteriorate. One hack invites another.
- **Deep vs shallow modules** (A Philosophy of Software Design) — a good module has a simple interface hiding complex implementation. If a PR adds complexity to the interface without adding capability, push back.
- **Tactical vs strategic** (Ousterhout) — is this PR a quick hack that adds tech debt, or a well-thought investment? Both have their place, but name it.

## Inspired By

- **Code Complete** — Steve McConnell
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **A Philosophy of Software Design** — John Ousterhout
