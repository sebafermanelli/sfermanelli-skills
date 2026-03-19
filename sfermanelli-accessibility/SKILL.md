---
name: sfermanelli-accessibility
description: Audit and fix accessibility issues in frontend code — semantic HTML, ARIA, keyboard navigation, screen readers, color contrast, WCAG compliance. Use this skill when the user asks about accessibility, a11y, WCAG, screen readers, keyboard navigation, or says things like "make this accessible", "a11y audit", "fix accessibility", "WCAG compliance", "add aria labels". Distinct from security-check (which audits for vulnerabilities) — this audits for inclusivity and WCAG compliance.
---

# Accessibility

Build interfaces that work for everyone. Accessibility is not a feature — it's a quality attribute like performance or security. An inaccessible app is a broken app for a significant portion of users.

> "The power of the Web is in its universality. Access by everyone regardless of disability is an essential aspect." — Tim Berners-Lee

## Golden Rule

**Use semantic HTML first, ARIA second, hacks never.** A `<button>` beats `<div role="button" tabindex="0" onkeydown="...">` every time. Reach for ARIA only when HTML semantics aren't enough.

---

## 1. Semantic HTML

The foundation of accessibility. Semantic elements carry meaning that assistive technologies understand natively — no ARIA needed.

### Use the Right Element

```tsx
// Bad — div soup (no semantics, no keyboard support, no screen reader info)
<div className="btn" onClick={handleSubmit}>Submit</div>
<div className="input-wrapper">
  <div className="label">Email</div>
  <div className="input" contentEditable />
</div>
<div className="link" onClick={() => navigate('/home')}>Home</div>

// Good — semantic elements (keyboard, focus, screen reader support for free)
<button type="submit" onClick={handleSubmit}>Submit</button>
<label>
  Email
  <input type="email" required />
</label>
<a href="/home">Home</a>
```

### What You Get For Free With Semantic HTML

| Element     | Keyboard     | Focus   | Screen Reader   | Behavior                  |
| ----------- | ------------ | ------- | --------------- | ------------------------- |
| `<button>`  | Enter, Space | Yes     | "button" role   | Click events on key press |
| `<a href>`  | Enter        | Yes     | "link" role     | Navigation                |
| `<input>`   | Tab          | Yes     | Role by type    | Native validation         |
| `<select>`  | Arrow keys   | Yes     | "combobox" role | Native dropdown           |
| `<details>` | Enter, Space | Yes     | "group" role    | Toggle behavior           |
| `<dialog>`  | Escape       | Managed | "dialog" role   | Focus trap, backdrop      |

### Heading Hierarchy

Headings create a document outline that screen reader users navigate by. They're not for styling.

```tsx
// Bad — skipping levels, using headings for visual styling
<h1>My App</h1>
<h4>Welcome back</h4>       {/* Skipped h2, h3 — broken outline */}
<h2 className="small-text">Details</h2>  {/* Using heading for font size */}

// Good — proper hierarchy, CSS handles visual styling
<h1>My App</h1>
<h2>Welcome back</h2>
<h3>Details</h3>

// Screen reader reads the outline as:
// [1] My App
//   [2] Welcome back
//     [3] Details
```

### Landmark Regions

Screen reader users jump between landmarks to navigate the page structure.

```tsx
// Minimum landmarks for a typical page
<body>
  <header>
    <nav aria-label="Main navigation">{/* Primary navigation */}</nav>
  </header>

  <main>
    {/* One per page — the primary content */}
    <h1>Page Title</h1>
    {/* Page content */}
  </main>

  <aside aria-label="Related content">{/* Sidebar, related links, ads */}</aside>

  <footer>{/* Site footer, secondary links, copyright */}</footer>
</body>
```

### Semantic Element Reference

| Element               | Use for                                  | Instead of                      |
| --------------------- | ---------------------------------------- | ------------------------------- |
| `<header>`            | Page or section header                   | `<div class="header">`          |
| `<nav>`               | Navigation links                         | `<div class="nav">`             |
| `<main>`              | Primary content (one per page)           | `<div class="main">`            |
| `<aside>`             | Tangentially related content             | `<div class="sidebar">`         |
| `<footer>`            | Page or section footer                   | `<div class="footer">`          |
| `<section>`           | Thematic grouping (must have heading)    | `<div class="section">`         |
| `<article>`           | Self-contained content (blog post, card) | `<div class="card">`            |
| `<figure>`            | Image/diagram with caption               | `<div class="image-container">` |
| `<time>`              | Date/time value                          | `<span>March 15</span>`         |
| `<mark>`              | Highlighted/relevant text                | `<span class="highlight">`      |
| `<details>/<summary>` | Expandable content                       | Custom accordion                |
| `<dialog>`            | Modal/dialog                             | Custom modal div                |
| `<output>`            | Result of a calculation                  | `<span id="result">`            |

---

## 2. ARIA

### The Five Rules of ARIA (W3C)

1. **Don't use ARIA if native HTML works** — `<button>` not `<div role="button">`
2. **Don't change native semantics** — don't put `role="heading"` on a `<button>`
3. **All interactive ARIA elements must be keyboard accessible**
4. **Don't use `role="presentation"` or `aria-hidden="true"` on focusable elements**
5. **All interactive elements must have an accessible name**

### ARIA Attribute Categories

| Category       | Attributes                                        | Purpose                     |
| -------------- | ------------------------------------------------- | --------------------------- |
| **Roles**      | `role="alert"`, `role="tab"`, `role="dialog"`     | Define what an element IS   |
| **States**     | `aria-expanded`, `aria-checked`, `aria-selected`  | Current condition (changes) |
| **Properties** | `aria-label`, `aria-describedby`, `aria-required` | Characteristics (static)    |

### Live Regions — Dynamic Content Announcements

```tsx
// Polite — wait for user to finish current task, then announce
<div aria-live="polite" aria-atomic="true">
  {itemCount} items in cart
</div>

// Assertive — interrupt immediately (use sparingly)
<div aria-live="assertive">
  Your session will expire in 2 minutes
</div>

// Role shortcuts
<div role="status">                    {/* Implicit aria-live="polite" */}
  Form submitted successfully
</div>

<div role="alert">                     {/* Implicit aria-live="assertive" */}
  Payment failed. Please try again.
</div>

// Log — append-only region (chat messages, activity log)
<div role="log" aria-live="polite">
  {messages.map(msg => <p key={msg.id}>{msg.text}</p>)}
</div>
```

### Form Accessibility

```tsx
// Rule: every input MUST have a label

// Method 1: Wrapping label (simplest)
<label>
  Email address
  <input type="email" required />
</label>

// Method 2: htmlFor/id association
<label htmlFor="email">Email address</label>
<input id="email" type="email" aria-required="true" />

// Method 3: aria-label (when visual label is absent — icon buttons, search)
<input type="search" aria-label="Search products" />

// Method 4: aria-labelledby (when the label is elsewhere in the DOM)
<h2 id="billing-heading">Billing Information</h2>
<input aria-labelledby="billing-heading" />

// Error states — connect error messages to the input
<label htmlFor="password">Password</label>
<input
  id="password"
  type="password"
  aria-invalid={!!error}
  aria-describedby={[
    error && 'password-error',
    'password-help'
  ].filter(Boolean).join(' ')}
  aria-required="true"
/>
<span id="password-help">Minimum 8 characters, one number</span>
{error && (
  <span id="password-error" role="alert" className="text-red-600">
    {error}
  </span>
)}

// Grouping related inputs
<fieldset>
  <legend>Shipping Address</legend>
  <label>Street <input name="street" /></label>
  <label>City <input name="city" /></label>
  <label>ZIP <input name="zip" /></label>
</fieldset>
```

### Common Widget Patterns (WAI-ARIA Authoring Practices)

#### Tabs

```tsx
function Tabs({ tabs }: { tabs: TabData[] }) {
  const [activeIndex, setActiveIndex] = useState(0)

  function handleKeyDown(e: KeyboardEvent) {
    let newIndex = activeIndex
    switch (e.key) {
      case 'ArrowRight':
        newIndex = (activeIndex + 1) % tabs.length
        break
      case 'ArrowLeft':
        newIndex = (activeIndex - 1 + tabs.length) % tabs.length
        break
      case 'Home':
        newIndex = 0
        break
      case 'End':
        newIndex = tabs.length - 1
        break
      default:
        return
    }
    setActiveIndex(newIndex)
    e.preventDefault()
  }

  return (
    <div>
      <div role="tablist" aria-label="Feature tabs" onKeyDown={handleKeyDown}>
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={i === activeIndex}
            aria-controls={`panel-${tab.id}`}
            tabIndex={i === activeIndex ? 0 : -1} // Roving tabindex
            onClick={() => setActiveIndex(i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={i !== activeIndex}
          tabIndex={0}
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

#### Dialog / Modal

```tsx
function AccessibleDialog({ isOpen, onClose, title, description, children }: DialogProps) {
  const dialogRef = useRef<HTMLDialogElement>(null)

  useEffect(() => {
    const dialog = dialogRef.current
    if (!dialog) return
    if (isOpen) {
      dialog.showModal() // Native focus trap + backdrop
    } else {
      dialog.close()
    }
  }, [isOpen])

  return (
    <dialog
      ref={dialogRef}
      aria-labelledby="dialog-title"
      aria-describedby="dialog-desc"
      onClose={onClose}
      onCancel={onClose} // Escape key triggers cancel event
    >
      <h2 id="dialog-title">{title}</h2>
      {description && <p id="dialog-desc">{description}</p>}
      {children}
      <button onClick={onClose} autoFocus>
        Close
      </button>
    </dialog>
  )
}

// Usage
;<AccessibleDialog
  isOpen={showDeleteConfirm}
  onClose={() => setShowDeleteConfirm(false)}
  title="Delete order?"
  description="This action cannot be undone. All order data will be permanently removed."
>
  <button onClick={handleDelete}>Delete permanently</button>
  <button onClick={() => setShowDeleteConfirm(false)} autoFocus>
    Cancel
  </button>
</AccessibleDialog>
```

#### Combobox / Autocomplete

```tsx
function Combobox({ options, onSelect, label }: ComboboxProps) {
  const [query, setQuery] = useState('')
  const [isOpen, setIsOpen] = useState(false)
  const [activeIndex, setActiveIndex] = useState(-1)
  const filtered = options.filter(o => o.label.toLowerCase().includes(query.toLowerCase()))

  function handleKeyDown(e: KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        setIsOpen(true)
        setActiveIndex(prev => Math.min(prev + 1, filtered.length - 1))
        break
      case 'ArrowUp':
        e.preventDefault()
        setActiveIndex(prev => Math.max(prev - 1, 0))
        break
      case 'Enter':
        if (activeIndex >= 0) {
          onSelect(filtered[activeIndex])
          setIsOpen(false)
        }
        break
      case 'Escape':
        setIsOpen(false)
        setActiveIndex(-1)
        break
    }
  }

  return (
    <div>
      <label htmlFor="combo-input">{label}</label>
      <input
        id="combo-input"
        role="combobox"
        aria-expanded={isOpen}
        aria-controls="combo-listbox"
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
        aria-autocomplete="list"
        value={query}
        onChange={e => {
          setQuery(e.target.value)
          setIsOpen(true)
        }}
        onKeyDown={handleKeyDown}
      />
      {isOpen && (
        <ul id="combo-listbox" role="listbox">
          {filtered.map((option, i) => (
            <li
              key={option.value}
              id={`option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => {
                onSelect(option)
                setIsOpen(false)
              }}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

---

## 3. Keyboard Navigation

### Focus Management Principles

```
1. Every interactive element MUST be reachable via Tab
2. Focus order MUST follow visual/reading order
3. Focus MUST be visible (never outline: none without a replacement)
4. No keyboard traps (except intentional modal traps with Escape exit)
5. Custom widgets use roving tabindex (one Tab stop, arrows to navigate within)
```

### Focus Trap Hook

```tsx
function useFocusTrap(ref: RefObject<HTMLElement>) {
  useEffect(() => {
    const element = ref.current
    if (!element) return

    const focusableSelector =
      'button:not([disabled]), [href], input:not([disabled]), select:not([disabled]), ' +
      'textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'

    const focusableElements = element.querySelectorAll(focusableSelector)
    const firstElement = focusableElements[0] as HTMLElement
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement

    // Save and restore focus
    const previouslyFocused = document.activeElement as HTMLElement

    function handleKeyDown(e: KeyboardEvent) {
      if (e.key !== 'Tab') return

      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault()
        lastElement.focus()
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault()
        firstElement.focus()
      }
    }

    element.addEventListener('keydown', handleKeyDown)
    firstElement?.focus()

    return () => {
      element.removeEventListener('keydown', handleKeyDown)
      previouslyFocused?.focus() // Restore focus on unmount
    }
  }, [ref])
}
```

### Roving Tabindex Pattern

For composite widgets where only one item is tabbable. Arrows move focus within the group.

```tsx
function useRovingTabindex<T extends HTMLElement>(
  items: RefObject<T>[],
  options: { orientation?: 'horizontal' | 'vertical'; loop?: boolean } = {}
) {
  const [focusedIndex, setFocusedIndex] = useState(0)
  const { orientation = 'horizontal', loop = true } = options

  const prevKey = orientation === 'horizontal' ? 'ArrowLeft' : 'ArrowUp'
  const nextKey = orientation === 'horizontal' ? 'ArrowRight' : 'ArrowDown'

  function handleKeyDown(e: KeyboardEvent) {
    let newIndex = focusedIndex

    switch (e.key) {
      case nextKey:
        newIndex = loop
          ? (focusedIndex + 1) % items.length
          : Math.min(focusedIndex + 1, items.length - 1)
        break
      case prevKey:
        newIndex = loop
          ? (focusedIndex - 1 + items.length) % items.length
          : Math.max(focusedIndex - 1, 0)
        break
      case 'Home':
        newIndex = 0
        break
      case 'End':
        newIndex = items.length - 1
        break
      default:
        return
    }

    e.preventDefault()
    setFocusedIndex(newIndex)
    items[newIndex].current?.focus()
  }

  return { focusedIndex, handleKeyDown }
}
```

### Keyboard Interaction Patterns (WAI-ARIA)

| Component   | Tab         | Arrow Keys     | Enter         | Space    | Escape     | Home/End   |
| ----------- | ----------- | -------------- | ------------- | -------- | ---------- | ---------- |
| Button      | Focus       | —              | Activate      | Activate | —          | —          |
| Link        | Focus       | —              | Navigate      | —        | —          | —          |
| Tabs        | To tab list | Switch tab     | —             | —        | —          | First/Last |
| Menu        | Open menu   | Navigate       | Select        | —        | Close      | First/Last |
| Dialog      | Within trap | —              | Submit?       | —        | Close      | —          |
| Combobox    | Focus input | Navigate list  | Select        | —        | Close list | —          |
| Tree        | Focus tree  | Navigate nodes | Toggle/Select | Toggle   | —          | First/Last |
| Slider      | Focus       | Adjust value   | —             | —        | —          | Min/Max    |
| Checkbox    | Focus       | —              | —             | Toggle   | —          | —          |
| Radio group | To group    | Select option  | —             | —        | —          | —          |

### Skip Links

```tsx
// First focusable element — lets keyboard users skip repetitive navigation
;<a
  href="#main-content"
  className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4
             focus:z-50 focus:bg-white focus:p-4 focus:shadow-lg focus:rounded"
>
  Skip to main content
</a>

{
  /* Navigation */
}

;<main id="main-content" tabIndex={-1}>
  {/* tabIndex={-1} makes it focusable but not in tab order */}
  <h1>Page Title</h1>
</main>
```

---

## 4. Visual Accessibility

### Color Contrast

```
WCAG AA Requirements (minimum for most apps):
┌─────────────────────────┬───────────────┐
│ Normal text (<18px)      │ 4.5:1 ratio   │
│ Large text (≥18px bold   │ 3:1 ratio     │
│   or ≥24px regular)      │               │
│ UI components & icons    │ 3:1 ratio     │
│ Focus indicators         │ 3:1 ratio     │
└─────────────────────────┴───────────────┘

WCAG AAA Requirements (enhanced):
┌─────────────────────────┬───────────────┐
│ Normal text              │ 7:1 ratio     │
│ Large text               │ 4.5:1 ratio   │
└─────────────────────────┴───────────────┘

Tools:
- Chrome DevTools → Inspect element → color picker shows contrast ratio
- Lighthouse accessibility audit
- axe browser extension
```

### Don't Rely on Color Alone (WCAG 1.4.1)

```tsx
// Bad — color is the only differentiator
<span style={{ color: error ? 'red' : 'green' }}>{status}</span>

<div className="chart">
  <div className="bar" style={{ backgroundColor: 'red' }} />  {/* Revenue */}
  <div className="bar" style={{ backgroundColor: 'green' }} />  {/* Profit */}
</div>

// Good — redundant coding: color + text + icon + pattern
<span style={{ color: error ? 'red' : 'green' }}>
  {error ? '✕ Error: ' : '✓ Success: '}{status}
</span>

<div className="chart">
  <div className="bar" style={{ backgroundColor: 'red', backgroundImage: 'url(diagonal-lines.svg)' }}>
    <span className="sr-only">Revenue</span>
  </div>
  <div className="bar" style={{ backgroundColor: 'green', backgroundImage: 'url(dots.svg)' }}>
    <span className="sr-only">Profit</span>
  </div>
</div>
```

### Focus Indicators

```css
/* Never do this — removes focus visibility for keyboard users */
*:focus {
  outline: none;
} /* NEVER */

/* Good — custom focus style that's visible on any background */
:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

/* Even better — high contrast ring */
:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.2);
}

/* Only show focus styles for keyboard users, not mouse clicks */
/* :focus-visible is the modern solution — no JS needed */
```

### Motion and Animation

```css
/* Respect user preferences — prefers-reduced-motion */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* In React/Angular — check programmatically */
```

```tsx
function usePrefersReducedMotion(): boolean {
  const [prefersReduced, setPrefersReduced] = useState(false)

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    setPrefersReduced(mediaQuery.matches)

    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches)
    mediaQuery.addEventListener('change', handler)
    return () => mediaQuery.removeEventListener('change', handler)
  }, [])

  return prefersReduced
}

// Usage
function AnimatedComponent() {
  const prefersReduced = usePrefersReducedMotion()
  return <motion.div animate={{ x: 100 }} transition={{ duration: prefersReduced ? 0 : 0.3 }} />
}
```

### Responsive Text and Zoom

```css
/* Use relative units — support up to 200% zoom (WCAG 1.4.4) */
font-size: 1rem; /* Good — scales with user preference */
font-size: 16px; /* Bad — doesn't scale */
font-size: clamp(1rem, 2.5vw, 1.5rem); /* Good — fluid and bounded */

/* Ensure content reflows at 400% zoom / 320px width (WCAG 1.4.10) */
/* No horizontal scrolling at 320px viewport width */
@media (max-width: 320px) {
  .container {
    flex-direction: column;
    overflow-x: hidden;
  }
}
```

---

## 5. Screen Reader Patterns

### Visually Hidden Content

```css
/* Content accessible to screen readers but not visible on screen */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Becomes visible on focus — for skip links */
.sr-only-focusable:focus {
  position: static;
  width: auto;
  height: auto;
  padding: inherit;
  margin: inherit;
  overflow: visible;
  clip: auto;
  white-space: normal;
}
```

### Icon Buttons

```tsx
// Icon-only buttons ALWAYS need accessible names

// Method 1: aria-label
<button aria-label="Close dialog" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>

// Method 2: Visually hidden text (better for translation)
<button onClick={onClose}>
  <XIcon aria-hidden="true" />
  <span className="sr-only">Close dialog</span>
</button>

// Method 3: Tooltip that doubles as label
<button aria-label="Delete order" title="Delete order" onClick={onDelete}>
  <TrashIcon aria-hidden="true" />
</button>

// Icon + text — icon is decorative, hide it from screen readers
<button onClick={onSave}>
  <SaveIcon aria-hidden="true" />
  Save changes
</button>
```

### Images

```tsx
// Rule: Every <img> must have an alt attribute

// Informative — describe the content/purpose
<img src="chart.png" alt="Revenue grew 25% in Q4 2024, reaching $1.2M" />

// Decorative — empty alt, hide from screen readers
<img src="decorative-border.png" alt="" />

// Functional — describe the action, not the image
<a href="/home">
  <img src="logo.png" alt="MyApp home page" />  {/* Not "company logo" */}
</a>

// Complex — provide extended description
<figure>
  <img
    src="architecture.png"
    alt="System architecture showing three interconnected services"
    aria-describedby="arch-desc"
  />
  <figcaption id="arch-desc">
    The architecture consists of three services: the API gateway handles
    authentication and routing, the order processor manages business logic,
    and the notification service sends emails and push notifications.
    Services communicate via a Redis message queue.
  </figcaption>
</figure>
```

### Tables

```tsx
// Data tables — always include headers with scope
<table>
  <caption>
    Order history for March 2024
    <span className="sr-only"> — sortable by clicking column headers</span>
  </caption>
  <thead>
    <tr>
      <th scope="col" aria-sort="none">Order ID</th>
      <th scope="col" aria-sort="descending">Date</th>
      <th scope="col" aria-sort="none">Total</th>
      <th scope="col">Status</th>
      <th scope="col"><span className="sr-only">Actions</span></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">#1234</th>
      <td><time dateTime="2024-03-15">Mar 15, 2024</time></td>
      <td>$49.99</td>
      <td>
        <span className="badge badge-green" role="status">Shipped</span>
      </td>
      <td>
        <button aria-label="View order #1234">View</button>
        <button aria-label="Cancel order #1234">Cancel</button>
      </td>
    </tr>
  </tbody>
</table>

// Layout tables — mark as presentational
<table role="presentation">
  {/* Only for visual layout, no data semantics */}
</table>
```

### Loading States

```tsx
// Announce loading to screen readers
function LoadingState({ isLoading, children }: { isLoading: boolean; children: ReactNode }) {
  return (
    <>
      <div aria-live="polite" aria-busy={isLoading}>
        {isLoading ? (
          <div role="status">
            <span className="sr-only">Loading...</span>
            <Spinner aria-hidden="true" />
          </div>
        ) : (
          children
        )}
      </div>
    </>
  )
}

// Skeleton screens — hide from screen readers, announce when loaded
function SkeletonLoader() {
  return (
    <div aria-hidden="true">
      <div className="skeleton h-4 w-3/4 mb-2" />
      <div className="skeleton h-4 w-1/2" />
    </div>
  )
}
```

### Toast / Notification Pattern

```tsx
// Toasts must be announced to screen readers
function ToastContainer({ toasts }: { toasts: Toast[] }) {
  return (
    <div aria-live="polite" aria-label="Notifications" className="fixed bottom-4 right-4">
      {toasts.map(toast => (
        <div
          key={toast.id}
          role={toast.type === 'error' ? 'alert' : 'status'}
          className={`toast toast-${toast.type}`}
        >
          <span>{toast.message}</span>
          <button aria-label={`Dismiss: ${toast.message}`} onClick={() => dismissToast(toast.id)}>
            <XIcon aria-hidden="true" />
          </button>
        </div>
      ))}
    </div>
  )
}
```

---

## 6. Testing Accessibility

### Automated Testing

```typescript
// 1. Development: @axe-core/react — real-time violations in console
if (process.env.NODE_ENV === 'development') {
  import('@axe-core/react').then((axe) => {
    axe.default(React, ReactDOM, 1000)
    // Violations appear in browser console with fix suggestions
  })
}

// 2. Unit tests: jest-axe — catch violations in CI
import { axe, toHaveNoViolations } from 'jest-axe'

expect.extend(toHaveNoViolations)

test('login form is accessible', async () => {
  const { container } = render(<LoginForm />)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})

test('modal is accessible when open', async () => {
  const { container } = render(<OrderDeleteModal isOpen={true} />)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})

// 3. E2E tests: Playwright + axe
import { test, expect } from '@playwright/test'
import AxeBuilder from '@axe-core/playwright'

test('order page is accessible', async ({ page }) => {
  await page.goto('/orders')
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze()
  expect(results.violations).toEqual([])
})

// 4. Keyboard navigation tests
test('can complete checkout with keyboard only', async ({ page }) => {
  await page.goto('/checkout')

  // Tab to first input
  await page.keyboard.press('Tab')
  await expect(page.locator('#email')).toBeFocused()

  // Fill form with keyboard
  await page.keyboard.type('user@example.com')
  await page.keyboard.press('Tab')
  await page.keyboard.type('John Doe')

  // Submit with Enter
  await page.keyboard.press('Tab')
  await page.keyboard.press('Enter')

  await expect(page.locator('[role="status"]')).toContainText('Order placed')
})
```

### Manual Testing Checklist

```markdown
## Keyboard Testing

- [ ] Tab through entire page — every interactive element reachable
- [ ] Focus order follows visual/reading order
- [ ] Focus indicator visible on EVERY focused element
- [ ] No keyboard traps (can always Tab/Escape away)
- [ ] Escape closes modals, dropdowns, tooltips
- [ ] Enter/Space activates buttons and links
- [ ] Arrow keys work in tabs, menus, sliders
- [ ] Skip link present and functional

## Screen Reader Testing (VoiceOver / NVDA)

- [ ] Page title announced on load
- [ ] Headings create logical outline (Rotor → Headings)
- [ ] All images have meaningful alt text (or are hidden)
- [ ] Form inputs announce their labels
- [ ] Error messages announced when they appear
- [ ] Dynamic content changes announced via live regions
- [ ] Links and buttons have unique, descriptive text
- [ ] Tables announce headers with data cells
- [ ] Landmarks present (navigation, main, footer)

## Visual Testing

- [ ] Text contrast ≥ 4.5:1 (use axe or DevTools)
- [ ] UI component contrast ≥ 3:1
- [ ] Content readable at 200% zoom
- [ ] No horizontal scroll at 320px width
- [ ] No info conveyed by color alone
- [ ] prefers-reduced-motion respected
- [ ] Focus indicators meet 3:1 contrast
- [ ] Dark mode maintains contrast ratios

## Content Testing

- [ ] Language attribute set on <html>
- [ ] Abbreviations expanded on first use or with <abbr>
- [ ] Error messages suggest how to fix the issue
- [ ] Time limits can be extended or disabled
```

---

## 7. WCAG 2.1 Quick Reference

### The Four Principles (POUR)

| Principle          | Meaning                                   | Key Guidelines                                        |
| ------------------ | ----------------------------------------- | ----------------------------------------------------- |
| **Perceivable**    | Users can perceive the content            | Alt text, captions, contrast, text resize             |
| **Operable**       | Users can operate the interface           | Keyboard access, enough time, no seizures, navigation |
| **Understandable** | Users can understand the content          | Readable, predictable, input assistance               |
| **Robust**         | Content works with assistive technologies | Valid HTML, name/role/value, status messages          |

### Level A (Minimum — legally required in many jurisdictions)

- All non-text content has text alternatives
- All functionality available from keyboard
- No keyboard traps
- Content doesn't flash more than 3 times/second
- Pages have descriptive titles
- Focus order is meaningful
- Link purpose clear from link text

### Level AA (Target for most applications)

- Color contrast meets 4.5:1 / 3:1
- Text resizable to 200% without loss of content
- Multiple ways to find pages (nav, search, sitemap)
- Consistent navigation and identification across pages
- Error identification with suggestions
- Labels or instructions for user input
- Status messages programmatically determinable

### Level AAA (Enhanced — aspirational, not typically required)

- 7:1 contrast ratio for text
- No images of text (except logos)
- Sign language for multimedia
- Extended audio descriptions
- Reading level assistance

---

## 8. Angular-Specific Accessibility

### Angular CDK A11y Module

```typescript
import { A11yModule, LiveAnnouncer, FocusTrapFactory } from '@angular/cdk/a11y'

@Component({
  selector: 'app-dialog',
  template: `
    <div cdkTrapFocus [cdkTrapFocusAutoCapture]="true">
      <h2>{{ title }}</h2>
      <ng-content></ng-content>
      <button (click)="close()">Close</button>
    </div>
  `,
})
export class DialogComponent {
  constructor(private liveAnnouncer: LiveAnnouncer) {}

  announce(message: string) {
    this.liveAnnouncer.announce(message, 'polite')
  }
}
```

---

## 9. Process

1. **Audit structure** — Semantic HTML, heading hierarchy, landmarks, page title
2. **Check interactions** — Tab through everything with keyboard only
3. **Verify ARIA** — Only where HTML semantics aren't enough, correctly applied
4. **Test contrast** — Browser DevTools, axe extension, Lighthouse
5. **Test with screen reader** — VoiceOver (Mac: Cmd+F5), NVDA (Windows)
6. **Run automated checks** — axe-core in dev, jest-axe in CI, Playwright in E2E
7. **Test responsive** — 200% zoom, 320px width, reduced motion

---

## Output

```
## Accessibility Audit

### Critical (must fix — WCAG A violations)
- [Issue] — [Location] — [WCAG criterion] — [Fix]

### Important (should fix — WCAG AA violations)
- [Issue] — [Location] — [WCAG criterion] — [Fix]

### Suggestions (best practices, AAA)
- [Issue] — [Location] — [Fix]

### Scores
- Keyboard navigable: ✓/✗
- Screen reader compatible: ✓/✗
- WCAG AA contrast: ✓/✗
- Automated scan (axe): X violations
- Lighthouse a11y score: X/100

### Fixed
- [What was fixed, WCAG criterion, how]
```

## Inspired By

- **WCAG 2.1 Guidelines** — W3C (Web Content Accessibility Guidelines)
- **WAI-ARIA Authoring Practices 1.2** — W3C (widget patterns and keyboard interaction)
- **Inclusive Components** — Heydon Pickering
- **A Web for Everyone** — Sarah Horton & Whitney Quesenbery
- **Accessibility for Everyone** — Laura Kalbag
- **Don't Make Me Think** — Steve Krug (usability principles that overlap with a11y)
- **Inclusive Design Patterns** — Heydon Pickering
- **Form Design Patterns** — Adam Silver
