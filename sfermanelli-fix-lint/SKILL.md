---
name: sfermanelli-fix-lint
description: Automatically fix linting and type errors across the codebase. Use this skill when the user asks to fix lint errors, fix ESLint, fix TypeScript errors, fix type errors, resolve warnings, clean up lint, or says things like "fix the red squiggles", "make it pass lint", "fix tsc errors", "resolve these warnings", "npm run lint has errors". Also triggers for "fix build errors" when they're type-related.
---

# Fix Lint

Fix linting and type errors efficiently — understand the root cause, apply the correct fix, not a band-aid.

## Process

1. **Run the linter/type checker** — detect the project's tools:
   - `npm run lint` or `npx eslint .`
   - `npx tsc --noEmit`
   - Check `package.json` scripts for custom lint commands
2. **Parse the errors** — group by type, not by file
3. **Fix root causes first** — one root cause often produces multiple errors
4. **Re-run to verify** — confirm zero errors after fixes
5. **Report** — what was fixed and why

## Fix Hierarchy (prefer higher over lower)

1. **Fix the actual code** — the code is wrong, fix the logic
2. **Add proper types** — missing type annotations, incorrect generics
3. **Handle the edge case** — null check, undefined guard, optional chaining
4. **Update the config** — rule is too strict or doesn't apply to this project
5. **Suppress with comment** — LAST RESORT, only with explanation

## Common Error Patterns

### TypeScript
| Error | Root Cause | Fix |
|-------|-----------|-----|
| `Type 'X' is not assignable to 'Y'` | Wrong type or missing conversion | Fix the type, add assertion if safe |
| `Property does not exist on type` | Missing field in type definition | Add to interface/type, or use optional chaining |
| `Cannot find module` | Missing import or package | Install package or fix import path |
| `Argument of type 'null' is not assignable` | Nullable not handled | Add null check or use `!` if guaranteed |
| `'X' is declared but never used` | Dead code | Remove it, or prefix with `_` if intentional |

### ESLint
| Error | Fix |
|-------|-----|
| `react-hooks/exhaustive-deps` | Add the missing dependency, or explain why it's intentional with `// eslint-disable-next-line` |
| `no-unused-vars` | Remove the variable, or prefix with `_` |
| `prefer-const` | Change `let` to `const` |
| `no-explicit-any` | Add proper type annotation |
| `react/no-unescaped-entities` | Use `&apos;` or `{'\''}`  |

## Rules for Suppressions

When you MUST suppress a lint rule:
```typescript
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- renderToBuffer types don't match React 19
const buffer = await renderToBuffer(element as any)
```

- Always use `next-line`, never `disable` for the whole file
- Always include a comment explaining WHY
- Never suppress security-related rules

## Anti-Patterns

- **`as any` everywhere** — fix the types properly
- **`@ts-ignore`** — almost never needed, use `@ts-expect-error` if you must (it breaks when the error is fixed, reminding you to remove it)
- **Deleting lint config rules** — unless the rule genuinely doesn't apply
- **Adding `| undefined` to everything** — handle the null case instead

## Output

After fixing:
```
## Lint Fix Summary

**Errors fixed:** 12
**Warnings fixed:** 3
**Suppressions added:** 1 (with justification)

### Changes:
- `src/actions/orders.ts` — added null check for `store` before accessing properties
- `src/components/ui/select.tsx` — fixed generic type parameter
- `src/lib/cart.ts` — removed unused import `useState`

### Verified:
- ✓ `npx tsc --noEmit` — 0 errors
- ✓ `npm run lint` — 0 errors, 0 warnings
```

## Key Principles

- **"Think of types as sets of values"** (Dan Vanderkam — Effective TypeScript) — a `string | number` is a union of two sets. Understanding this makes type errors intuitive.
- **Use type narrowing over assertions** (Effective TypeScript) — `if (x !== null)` is safer than `x!`. Narrowing is verified by the compiler; assertions are trust-me promises.
- **Error handling is one thing** (Robert C. Martin — Clean Code) — a function that handles errors should do nothing else. Don't mix error handling with business logic.
- **"Improving quality reduces development costs"** (Steve McConnell — Code Complete) — fixing lint/type errors early prevents cascading bugs later.

## Inspired By

- **Effective TypeScript** — Dan Vanderkam
- **Clean Code** — Robert C. Martin
- **Code Complete** — Steve McConnell
