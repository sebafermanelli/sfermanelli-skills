---
name: sfermanelli-explain-code
description: Explain code clearly at any level of detail. Use this skill when the user asks to explain code, understand a function, walk through logic, asks "what does this do", "how does this work", "explain this to me", "I don't understand this", or wants a codebase overview. Also triggers for "break this down", "what's happening here", "trace this flow", or "ELI5 this code".
---

# Explain Code

Explain code so it actually makes sense — not by restating what each line does, but by building understanding of WHY the code exists and HOW it fits into the bigger picture.

## Golden Rule

**Explain at the right level of abstraction.** Don't explain `const x = 5` means "assign 5 to x". Explain WHY it's 5 and not 3, what role this value plays, and what breaks if you change it.

---

## 1. How to Explain

### Start with the What (one sentence)

What does this code accomplish? Not how — what. If you can't explain it in one sentence, break it into smaller pieces first.

### Explain the Why

Why does this code exist? What problem does it solve? What would go wrong without it? This is the most important part — code without context is just syntax.

### Walk Through the How

Trace the logic at the right level of abstraction. Focus on the decisions: why this data structure, why this algorithm, why this pattern.

### Connect to the Bigger Picture

How does this code relate to the rest of the system? What calls it? What does it call? Where does the data come from and where does it go?

---

## 2. Adjust to the Audience

Read context cues from the user:

| Cue                                   | Level                   | Approach                                                |
| ------------------------------------- | ----------------------- | ------------------------------------------------------- |
| "ELI5", "I'm new"                     | Beginner                | Use analogies, avoid jargon, explain concepts           |
| "Walk me through this"                | Intermediate            | Explain the flow and key decisions                      |
| "What's the purpose of this pattern?" | Advanced                | Explain the architecture choice and trade-offs          |
| No context cues                       | Default to intermediate | Explain enough to be useful without being condescending |

---

## 3. Explanation Formats

### For a Single Function/Component

```
## `functionName`

**What it does:** [one sentence]

**Why it exists:** [the problem it solves]

**How it works:**
1. [step 1 — what and why]
2. [step 2 — what and why]
3. [step 3 — what and why]

**Key details:**
- [non-obvious behavior, edge cases, gotchas]

**Used by:** [what calls this, where it fits in the flow]
```

### For a Flow/Feature

```
## [Feature Name] Flow

**Overview:** [what the feature does from the user's perspective]

**Flow:**
User action → [component] → [action/API] → [database] → [response] → [UI update]

**Key files:**
- `path/to/file.ts` — [role in the flow]
- `path/to/other.ts` — [role in the flow]

**Important decisions:**
- [why X pattern was chosen over Y]
- [why this is server-side vs client-side]
```

### For a Codebase Overview

```
## Architecture Overview

**What this project does:** [one paragraph]

**Tech stack:** [framework, DB, auth, etc.]

**Directory structure:**
src/
  domain/       — [what lives here]
  application/  — [what lives here]
  infrastructure/ — [what lives here]

**Data flow:** [how data moves through the system]

**Key patterns:**
- [pattern 1 and why it's used]
- [pattern 2 and why it's used]
```

---

## 4. Explaining Common Patterns

When explaining code that uses well-known patterns, name the pattern and explain its purpose:

### Dependency Injection

```
"This class receives its dependencies (the database, the logger) through the constructor
instead of creating them internally. This is called Dependency Injection — it means you
can swap out the real database for a test double when testing, without changing the class."
```

### Observer / Events

```
"When an order is placed, the system doesn't directly call sendEmail() and updateInventory().
Instead, it emits an 'order.placed' event, and separate listeners handle email and inventory
independently. This is the Observer pattern — it means adding a new side effect (like analytics)
doesn't require changing the order code at all."
```

### Middleware Pipeline

```
"Each request passes through a chain of functions: first auth checks who you are, then
rate limiting checks how often you're calling, then validation checks the data, and
finally the handler processes the request. Each middleware can either pass the request
to the next one or reject it. Think of it like airport security — multiple checkpoints,
and you must pass all of them."
```

### Repository Pattern

```
"The OrderRepository is an interface that says 'I can find orders and save orders' without
specifying HOW. The PrismaOrderRepository is one implementation that uses Prisma/PostgreSQL.
You could swap it for a MongoOrderRepository or an InMemoryOrderRepository without changing
any business logic. It separates 'what the app needs' from 'how the database works'."
```

---

## 5. Using Diagrams

For complex flows, ASCII diagrams are more effective than paragraphs:

### Sequence Diagram

```
User        Client        API           Database
  |            |            |              |
  |--click---->|            |              |
  |            |--POST /orders-->          |
  |            |            |--INSERT----->|
  |            |            |<--order row--|
  |            |            |--emit event->|  (async)
  |            |<--201 Created--|          |
  |<--show confirmation--|  |              |
```

### Data Flow

```
[Browser] → POST /api/orders
              ↓
         [Middleware: Auth] → 401 if no token
              ↓
         [Middleware: Validate] → 400 if invalid body
              ↓
         [Handler: createOrder]
              ↓
         [Service: OrderService.create]
              ├─→ [Repository: save to DB]
              ├─→ [EventBus: emit OrderPlaced]
              │     ├─→ [EmailService: send confirmation]
              │     └─→ [InventoryService: reserve stock]
              └─→ return OrderDto
```

### State Machine

```
           place()        pay()         ship()
  DRAFT ──────────→ PLACED ────────→ PAID ────────→ SHIPPED ──→ DELIVERED
    │                  │
    │  cancel()        │  cancel()
    └──────→ CANCELLED ←──────┘
```

### Architecture

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Next.js   │────→│   NestJS     │────→│  PostgreSQL  │
│  (Frontend) │     │   (API)      │     │  (Database)  │
└─────────────┘     └──────┬───────┘     └──────────────┘
                           │
                    ┌──────┴───────┐
                    │    Redis     │
                    │  (Cache/Queue)│
                    └──────────────┘
```

---

## 6. Principles

- **Don't just restate the code in English** — "this loop iterates over the array" adds zero value. Explain WHY it iterates and WHAT it's building.
- **Use analogies for complex concepts** — "RLS policies work like a bouncer at a club — they check your ID before letting you see the data"
- **Point out the non-obvious** — the interesting part is usually not what the code does, but why it does it THIS way
- **Mention gotchas** — subtle bug risks, performance considerations, "this looks wrong but is actually correct"
- **Name the patterns** — when code follows a well-known pattern, name it. It gives the reader a mental shortcut.
- **Complexity = anything that makes code hard to understand** (Ousterhout) — identify what creates cognitive load: obscure naming, deep nesting, implicit dependencies
- **Abstraction barriers** (SICP) — explain systems in layers. Each layer has a clear interface and hides its internals.
- **Minimize time to understanding** (Art of Readable Code) — "Code should be written to minimize the time it would take someone else to understand it"

## Inspired By

- **The Art of Readable Code** — Dustin Boswell & Trevor Foucher
- **A Philosophy of Software Design** — John Ousterhout
- **Structure and Interpretation of Computer Programs** — Abelson & Sussman
- **Refactoring** — Martin Fowler (for naming patterns)
- **Design Patterns** — Gang of Four (for pattern vocabulary)
