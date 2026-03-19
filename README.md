# sfermanelli-skills

Claude Code skills for software engineering — reusable across any project.

## Skills

| Skill | Trigger | What it does |
|-------|---------|-------------|
| **write-tests** | "write tests", "add coverage" | Generate unit, integration, e2e tests |
| **review-pr** | "review PR", "check my changes" | Senior engineer code review |
| **explain-code** | "what does this do", "explain this" | Clear explanations at any level |
| **fix-lint** | "fix lint", "fix types" | Fix linting and TypeScript errors |
| **clean-code** | "clean this up", "improve readability" | Naming, functions, comments, formatting |
| **debug** | "this is broken", "I'm getting an error" | Root cause investigation |
| **refactor** | "refactor", "this file is too big" | Restructure without changing behavior |
| **security-check** | "is this secure", "security audit" | OWASP-based vulnerability audit |
| **performance** | "this is slow", "optimize" | Find and fix bottlenecks |

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

## Inspired by

- Clean Code — Robert C. Martin
- Refactoring — Martin Fowler
- The Pragmatic Programmer — Hunt & Thomas
- A Philosophy of Software Design — John Ousterhout
- Effective TypeScript — Dan Vanderkam
- Designing Data-Intensive Applications — Martin Kleppmann
- OWASP Top 10
- and more (see each SKILL.md for full references)

## License

MIT
