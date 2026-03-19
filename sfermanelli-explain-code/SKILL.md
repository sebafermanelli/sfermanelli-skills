---
name: sfermanelli-explain-code
description: Explain code clearly at any level of detail. Use this skill when the user asks to explain code, understand a function, walk through logic, asks "what does this do", "how does this work", "explain this to me", "I don't understand this", or wants a codebase overview. Also triggers for "break this down", "what's happening here", "trace this flow", or "ELI5 this code".
---

# Explain Code

Explain code so it actually makes sense — not by restating what each line does, but by building understanding of WHY the code exists and HOW it fits into the bigger picture.

## How to Explain

### 1. Start with the What (one sentence)
What does this code accomplish? Not how — what. If you can't explain it in one sentence, break it into smaller pieces first.

### 2. Explain the Why
Why does this code exist? What problem does it solve? What would go wrong without it? This is the most important part — code without context is just syntax.

### 3. Walk Through the How
Now trace the logic, but at the right level of abstraction. Don't explain `const x = 5` means "assign 5 to x". Focus on the decisions: why this data structure, why this algorithm, why this pattern.

### 4. Connect to the Bigger Picture
How does this code relate to the rest of the system? What calls it? What does it call? Where does the data come from and where does it go?

## Adjust to the Audience

Read context cues from the user:
- **"ELI5"** or **"I'm new to this"** → use analogies, avoid jargon, explain concepts
- **"Walk me through this"** → mid-level, explain the flow and key decisions
- **"What's the purpose of this pattern?"** → they know code, explain the architecture choice
- **No context cues** → default to mid-level, explain enough to be useful without being condescending

## Explanation Formats

### For a single function/component:
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

### For a flow/feature:
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

### For a codebase overview:
```
## Architecture Overview

**What this project does:** [one paragraph]

**Tech stack:** [framework, DB, auth, etc.]

**Directory structure:**
- `src/app/` — [what lives here]
- `src/components/` — [what lives here]
- `src/actions/` — [what lives here]

**Data flow:** [how data moves through the system]

**Key patterns:**
- [pattern 1 and why it's used]
- [pattern 2 and why it's used]
```

## Principles

- **Don't just restate the code in English** — "this loop iterates over the array" adds zero value. Explain WHY it iterates and WHAT it's building.
- **Use analogies for complex concepts** — "RLS policies work like a bouncer at a club — they check your ID (auth token) before letting you see the data"
- **Point out the non-obvious** — the interesting part is usually not what the code does, but why it does it THIS way
- **Mention gotchas** — if there's a subtle bug risk, a performance consideration, or a "this looks wrong but is actually correct" situation, call it out
- **Use diagrams when helpful** — ASCII flow diagrams for complex flows, tree structures for hierarchies
- **Complexity = anything that makes code hard to understand** (Ousterhout) — identify what creates cognitive load: obscure naming, deep nesting, implicit dependencies, unknown unknowns
- **Abstraction barriers** (SICP) — explain systems in layers. Each layer has a clear interface to the layer above and hides its internals.
- **Minimize time to understanding** (The Art of Readable Code) — "Code should be written to minimize the time it would take someone else to understand it"

## Inspired By

- **The Art of Readable Code** — Dustin Boswell & Trevor Foucher
- **A Philosophy of Software Design** — John Ousterhout
- **Structure and Interpretation of Computer Programs** — Abelson & Sussman
