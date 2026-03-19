---
name: sfermanelli-refactor
description: Restructure code to improve architecture without changing behavior. Use this skill when the user asks to refactor, restructure, reorganize, extract components, split files, move logic, reduce duplication, or improve architecture. Triggers for "this file is too big", "extract this into a component", "split this", "DRY this up", "reduce coupling", "improve the architecture", "this is getting complex". Different from clean-code (cosmetic) — this changes code structure.
---

# Refactor

Restructure code to be better organized, more maintainable, and easier to extend — without changing what it does.

## Golden Rule

**Refactoring changes structure, not behavior.** If the app works differently after your refactoring, you introduced a bug. Every refactoring step should be verifiable: run the tests, run the type checker, manually verify the feature still works.

## When to Refactor

- A file is over 300 lines and doing multiple things
- You're copying code between files instead of sharing it
- Changing one feature requires touching 5+ files
- A component has more than 5 boolean props
- A function has more than 4 parameters
- The same logic appears in 3+ places

## Refactoring Patterns

### Extract Component
When a part of a component has its own responsibility:

```typescript
// Before: 200-line component with inline form
export function StoreSettings({ store }) {
  // ...100 lines of form state and handlers...
  return (
    <div>
      {/* ...50 lines of store info... */}
      {/* ...50 lines of form... */}
    </div>
  )
}

// After: two focused components
export function StoreSettings({ store }) {
  return (
    <div>
      <StoreInfo store={store} />
      <StoreSettingsForm store={store} />
    </div>
  )
}
```

### Extract Utility
When logic is reused across files:

```typescript
// Before: same validation in 3 files
if (!email || !email.includes('@')) throw new Error('Invalid email')

// After: shared utility
// lib/validation.ts
export function validateEmail(email: string): boolean {
  return !!email && email.includes('@')
}
```

### Extract Hook
When component state logic is complex or reusable:

```typescript
// Before: 30 lines of pagination state in component
const [page, setPage] = useState(1)
const [total, setTotal] = useState(0)
// ...fetch logic, page calculations...

// After: custom hook
const { page, total, nextPage, prevPage } = usePagination({ pageSize: 20 })
```

### Simplify Conditionals
```typescript
// Before: nested ifs
if (user) {
  if (user.isAdmin) {
    if (store.ownerId === user.id) {
      // do thing
    }
  }
}

// After: early returns
if (!user) return
if (!user.isAdmin) return
if (store.ownerId !== user.id) return
// do thing
```

### Collocate Related Code
Move code closer to where it's used:
- Types next to the functions that use them
- Constants next to the module that needs them
- Helper functions in the same file if only used there

### Split by Concern
When a file does too many things:
```
// Before
actions/stores.ts (800 lines: CRUD + geocoding + schedules + fees)

// After
actions/stores.ts (create, update, delete)
actions/store-schedules.ts (schedule management)
actions/store-fees.ts (fee calculations)
```

## Process

1. **Identify the smell** — what's wrong with the current structure?
2. **Plan the change** — describe what moves where, check that nothing breaks conceptually
3. **Make one change at a time** — don't do 5 refactorings in one step
4. **Verify after each step** — `npx tsc --noEmit`, run tests if they exist
5. **Update imports** — make sure nothing points to the old location

## Anti-Patterns

- **Premature abstraction** — don't abstract until you have 3+ concrete cases
- **Over-engineering** — a simple function doesn't need a class, interface, factory, and strategy pattern
- **Refactoring under pressure** — don't refactor while fixing a production bug
- **Big bang refactoring** — prefer small, incremental changes over rewriting everything at once
- **Moving code without understanding it** — read first, move second

## Output

```markdown
## Refactoring Summary

**Goal:** [what structural problem was solved]

### Changes:
1. Extracted `StoreScheduleForm` from `StoreSettingsForm` (was 400 lines, now 180 + 220)
2. Created `lib/store-validation.ts` for shared validation logic (used in 3 actions)
3. Moved `calculateFees` to `lib/fees.ts` (was inline in actions/orders.ts)

### Files:
- `src/components/stores/store-settings-form.tsx` — split into form + schedule sections
- `src/components/stores/store-schedule-form.tsx` — new, extracted
- `src/lib/store-validation.ts` — new, shared validation
- `src/lib/fees.ts` — moved from inline

### Verified:
- ✓ `npx tsc --noEmit` — 0 errors
- ✓ All existing functionality preserved
- ✓ No import changes needed in consuming files (re-exported from original location)
```

## Code Smells Catalog (Martin Fowler — Refactoring)

Recognize these smells as signals that refactoring is needed:

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| **Long Method** | >30 lines, scrolling needed | Extract Method |
| **Large Class** | >300 lines, multiple responsibilities | Extract Class |
| **Feature Envy** | Method uses more data from another class | Move Method |
| **Data Clumps** | Same group of fields appear together | Introduce Parameter Object |
| **Primitive Obsession** | Using strings/numbers for domain concepts | Replace with Value Object |
| **Switch Statements** | Same switch in multiple places | Replace with Polymorphism |
| **Speculative Generality** | Abstractions with only one implementation | Inline / Remove |
| **Middle Man** | Class that only delegates | Remove Middle Man |

## Key Principles

### Deep Modules (John Ousterhout — A Philosophy of Software Design)
A good module has a **simple interface** hiding **complex implementation**. If your refactoring makes the interface more complex than what it hides, you're going the wrong direction. "The best modules provide powerful functionality yet have simple interfaces."

### Working with Legacy Code (Michael Feathers)
When refactoring untested code:
- **Sprout Method** — write new functionality in a new method, call it from the old code. Don't modify the old code.
- **Wrap Method** — create a new method with the old name, rename the old one, call both from the wrapper.
- **Characterize first** — write tests that capture current behavior before changing anything.

"Legacy code is code without tests." — Michael Feathers

## Inspired By

- **Refactoring** — Martin Fowler
- **A Philosophy of Software Design** — John Ousterhout
- **Working Effectively with Legacy Code** — Michael Feathers
- **Clean Code** — Robert C. Martin
