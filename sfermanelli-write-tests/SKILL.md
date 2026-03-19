---
name: sfermanelli-write-tests
description: Generate well-structured, production-quality tests for any codebase. Use this skill whenever the user asks to write tests, add test coverage, create unit/integration/e2e tests, test a function/component/API route, or mentions testing in any form — even casually like "this needs tests" or "can you test this". Also triggers for "add coverage", "write specs", "test this flow", or "make sure this works".
---

# Write Tests

Generate clean, maintainable, production-quality tests that actually catch bugs — not tests that just exist to inflate coverage numbers.

## Philosophy

Good tests have three properties:
1. **They break when the code is wrong** — if you can change the implementation to be buggy and the test still passes, the test is useless
2. **They don't break when the code is right** — brittle tests that fail on every refactor waste more time than they save
3. **They document behavior** — reading the test should tell you what the code does without reading the implementation

## Before Writing Tests

1. **Detect the project's test stack** — check `package.json`, `vitest.config.*`, `jest.config.*`, `playwright.config.*`, `tsconfig.json`, existing test files. Don't assume — detect.
2. **Find existing test patterns** — look at `**/*.test.*`, `**/*.spec.*`, `__tests__/` directories. Match the project's conventions for file naming, directory structure, imports, and assertion style.
3. **Understand what you're testing** — read the source code. Identify inputs, outputs, side effects, error paths, and edge cases. Don't write tests for code you don't understand.

## Test Structure

Use the Arrange-Act-Assert pattern. Every test should be readable top-to-bottom:

```typescript
it('returns the discounted price when a valid coupon is applied', () => {
  // Arrange
  const product = createProduct({ price: 100 })
  const coupon = createCoupon({ discount: 20 })

  // Act
  const result = applyDiscount(product, coupon)

  // Assert
  expect(result.finalPrice).toBe(80)
})
```

## Naming Convention

Test names should describe behavior, not implementation:

**Bad:** `it('calls calculateFee')`
**Good:** `it('charges a 10% platform fee on the subtotal')`

**Bad:** `it('returns true')`
**Good:** `it('allows checkout when all items are in stock')`

Use `describe` blocks to group by feature or function, not by method name.

## What to Test

### Unit Tests (functions, utils, business logic)
- Happy path with typical inputs
- Edge cases: empty arrays, null/undefined, zero, negative numbers, boundary values
- Error cases: invalid inputs, missing required fields
- Return value AND side effects (if any)

### Integration Tests (API routes, server actions, DB operations)
- Request → response cycle with real or realistic data
- Authentication/authorization: test both allowed and denied
- Error responses: 400, 401, 403, 404, 500
- Database state changes (if applicable)

### Component Tests (React/UI components)
- Renders without crashing
- Displays correct content based on props
- User interactions (click, type, submit) produce expected results
- Conditional rendering (loading, error, empty states)
- Accessibility: roles, labels, keyboard navigation

### E2E Tests (Playwright/Cypress)
- Critical user flows: signup → action → result
- Navigation and routing
- Form submissions with validation
- Responsive behavior if relevant

## Framework-Specific Patterns

### Vitest / Jest
```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest' // or jest globals

describe('calculateOrderFees', () => {
  it('applies platform fee as percentage of subtotal', () => {
    const result = calculateOrderFees(1000, 10, 50)
    expect(result.platformFeeAmount).toBe(100)
    expect(result.totalCharged).toBe(1050)
  })

  it('handles zero subtotal', () => {
    const result = calculateOrderFees(0, 10, 50)
    expect(result.platformFeeAmount).toBe(0)
    expect(result.totalCharged).toBe(50) // fixed fee still applies
  })
})
```

### React Testing Library
```typescript
import { render, screen, fireEvent } from '@testing-library/react'

it('disables submit button when form is incomplete', () => {
  render(<CheckoutForm />)
  const button = screen.getByRole('button', { name: /confirm/i })
  expect(button).toBeDisabled()
})
```

### Playwright
```typescript
import { test, expect } from '@playwright/test'

test('user can add product to cart', async ({ page }) => {
  await page.goto('/stores/test-store?start=2026-01-01&end=2026-01-07')
  await page.getByRole('button', { name: /add to cart/i }).first().click()
  await expect(page.getByText(/in cart/i)).toBeVisible()
})
```

## Mocking Guidelines

- **Mock external services** (APIs, email, payment), never the code under test
- **Don't mock the database** in integration tests — use a test database or in-memory alternative
- **Mock sparingly** — every mock is a place where your test diverges from reality
- **Type your mocks** — `vi.fn<Parameters<typeof sendEmail>, ReturnType<typeof sendEmail>>()`
- **Reset mocks between tests** — use `beforeEach(() => vi.clearAllMocks())`

## Output Format

When generating tests:

1. Create the test file following the project's naming convention (`.test.ts`, `.spec.ts`, etc.)
2. Import only what's needed
3. Group related tests with `describe`
4. Include a comment at the top explaining what's being tested if the file covers complex logic
5. After writing, run the tests to verify they pass: `npx vitest run <file>` or `npx jest <file>`
6. Report results: how many tests, how many pass, coverage if available

## Common Mistakes to Avoid

- **Testing implementation details** — don't assert on internal state, private methods, or the number of times a function was called (unless that IS the behavior)
- **Copy-paste tests with minimal changes** — each test should test something meaningfully different
- **Ignoring async behavior** — always `await` async operations, use `waitFor` in component tests
- **Snapshot testing as a crutch** — snapshots are fine for UI regression, but they don't verify behavior
- **Testing framework code** — don't test that React renders a div or that Express routes work. Test YOUR code.

## Key Principles from the Masters

### Red-Green-Refactor (Kent Beck — TDD by Example)
1. **Red** — write a failing test that defines what you want
2. **Green** — write the simplest code that makes it pass
3. **Refactor** — clean up without changing behavior
This cycle forces you to think about the API before the implementation.

### Characterization Tests (Michael Feathers — Working Effectively with Legacy Code)
When working with existing code that has no tests, write **characterization tests** first — tests that capture the current behavior, even if that behavior has bugs. This gives you a safety net before making changes. "A characterization test is a test that characterizes the actual behavior of a piece of code."

### Test Doubles Taxonomy (Gerard Meszaros — xUnit Test Patterns)
Use the right double for the job:
- **Dummy** — passed but never used (just fills a parameter)
- **Stub** — returns predetermined answers
- **Spy** — records calls for later verification
- **Mock** — pre-programmed with expectations
- **Fake** — working implementation but simplified (in-memory DB)

### Find Bugs Once (Hunt & Thomas — The Pragmatic Programmer)
"Once a human tester finds a bug, it should be the last time a human tester finds that bug. Automatic tests should check for it from then on." Every bug report becomes a test case.

## Inspired By

- **TDD by Example** — Kent Beck
- **Working Effectively with Legacy Code** — Michael Feathers
- **xUnit Test Patterns** — Gerard Meszaros
- **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- **Code Complete** — Steve McConnell
