# sfermanelli-skills

Claude Code skills for software engineering — reusable across any project.

15 skills covering the full development lifecycle: from architecture design to deployment, with TypeScript examples, DDD patterns, and references from recognized software engineering books.

## Skills

### Code Quality

| Skill           | Trigger                                | What it does                                                                       |
| --------------- | -------------------------------------- | ---------------------------------------------------------------------------------- |
| **clean-code**  | "clean this up", "improve readability" | Naming, functions, comments, formatting, Law of Demeter, guard clauses             |
| **refactor**    | "refactor", "this file is too big"     | Extract Method/Class, Replace Conditional with Polymorphism, Value Objects         |
| **fix-lint**    | "fix lint", "fix types"                | TypeScript errors, ESLint, type narrowing, strict migration, Prisma types          |
| **write-tests** | "write tests", "add coverage"          | Unit/integration/e2e, factories, NestJS TestingModule, Angular TestBed, Playwright |

### Design & Architecture

| Skill              | Trigger                                 | What it does                                                                    |
| ------------------ | --------------------------------------- | ------------------------------------------------------------------------------- |
| **architect**      | "design this", "how should I structure" | SOLID, DDD (aggregates, value objects, events), design patterns (GoF), ADRs     |
| **api-design**     | "design the API", "create endpoints"    | REST, Zod validation, DTOs, CQRS, idempotency, rate limiting, versioning        |
| **error-handling** | "handle errors", "add error boundary"   | Error hierarchies, Result pattern, circuit breaker, retry, graceful degradation |

### Review & Security

| Skill              | Trigger                            | What it does                                                         |
| ------------------ | ---------------------------------- | -------------------------------------------------------------------- |
| **review-pr**      | "review PR", "check my changes"    | Conventional comments, checklists by change type, PR size guidelines |
| **security-check** | "is this secure", "security audit" | OWASP Top 10, XSS/SQLi/IDOR examples, JWT, CORS, CSP, RLS policies   |

### Performance & Operations

| Skill           | Trigger                                  | What it does                                                                   |
| --------------- | ---------------------------------------- | ------------------------------------------------------------------------------ |
| **performance** | "this is slow", "optimize"               | Web Vitals, EXPLAIN ANALYZE, N+1, Angular OnPush, NestJS caching, memory leaks |
| **debug**       | "this is broken", "I'm getting an error" | DevTools, error decoder, async debugging, Angular/NestJS/Next.js patterns      |
| **migrate**     | "upgrade to v5", "migrate from X to Y"   | Expand-contract, Prisma migrations, dual-write, framework upgrades             |

### Documentation & Accessibility

| Skill             | Trigger                              | What it does                                                                |
| ----------------- | ------------------------------------ | --------------------------------------------------------------------------- |
| **write-docs**    | "document this", "add JSDoc"         | TSDoc, README structure, ADRs, Divio's 4 doc types, OpenAPI/Swagger         |
| **explain-code**  | "what does this do", "explain this"  | Layered explanations, pattern naming, ASCII diagrams                        |
| **accessibility** | "make this accessible", "a11y audit" | WCAG, ARIA widgets (tabs, combobox, dialog), keyboard patterns, axe testing |

## Install

```bash
# Clone and copy to your project
git clone https://github.com/sebafermanelli/sfermanelli-skills.git /tmp/sfermanelli-skills
cp -r /tmp/sfermanelli-skills/sfermanelli-* /your-project/.claude/skills/
rm -rf /tmp/sfermanelli-skills
```

Or as a one-liner:

```bash
git clone https://github.com/sebafermanelli/sfermanelli-skills.git /tmp/sf-skills && cp -r /tmp/sf-skills/sfermanelli-* .claude/skills/ && rm -rf /tmp/sf-skills
```

## Stacks Covered

Examples are in TypeScript and designed to be framework-agnostic, with specific patterns for:

- **Next.js** — Server/Client components, caching, middleware, App Router
- **NestJS** — Modules, guards, interceptors, pipes, TestingModule
- **Angular** — OnPush, TestBed, CDK a11y, RxJS patterns
- **Supabase** — RLS policies, auth, real-time
- **Prisma** — Migrations, type-safe queries, relation loading

## Inspired By

- **Clean Code** / **Clean Architecture** — Robert C. Martin
- **Refactoring** — Martin Fowler
- **A Philosophy of Software Design** — John Ousterhout
- **Domain-Driven Design** — Eric Evans
- **Design Patterns** — Gang of Four
- **The Pragmatic Programmer** — Hunt & Thomas
- **Effective TypeScript** — Dan Vanderkam
- **Designing Data-Intensive Applications** — Martin Kleppmann
- **Release It!** — Michael Nygard
- **Working Effectively with Legacy Code** — Michael Feathers
- **OWASP Top 10** / **Secure by Design**
- **WCAG 2.1** / **WAI-ARIA Authoring Practices**
- and more (see each SKILL.md for full references)

## License

MIT
