# Rules Files

Use `.claude/rules/<name>.md` with `paths:` frontmatter for conventions that apply to a specific area of the codebase — they load automatically only when matching files are touched.

### Key Concepts

- Rules with `paths:` glob frontmatter load **only** when matching tool inputs occur — keeps per-turn context lean
- Patterns evaluate against tool inputs, not the active editor file — broad patterns load on more turns than expected
- Overlapping `paths:` is fine, but conflicting rules compose unpredictably — keep rules additive
- Each rule should stay under ~30 lines; long bodies inflate context whenever they trigger

### How to

- Create one rule file per concern with a clear `paths:` glob
- Use `**/` for cross-directory patterns; `src/api/*` won't match `src/api/users/handler.ts` (use `src/api/**`)
- Focus rule bodies on what would otherwise be repeatedly mistaken — not exhaustive style guides
- For test conventions spanning the codebase, use `paths: ["**/*.test.ts"]` instead of per-directory CLAUDE.md duplication

```markdown
<!-- .claude/rules/components.md -->
---
name: components
description: React component conventions
paths: ["src/components/**"]
---
Functional components only · hooks at top · co-locate styles in `*.module.css`.

<!-- .claude/rules/api.md -->
---
name: api
description: Express handler conventions
paths: ["src/api/**"]
---
Validate input with zod · return `{data, error}` envelope · log with `req.log`.

<!-- .claude/rules/db.md -->
---
name: db
description: Database model conventions
paths: ["src/db/**"]
---
Models are immutable types · timestamps in UTC · no business logic in models.

<!-- .claude/rules/tests.md -->
---
name: tests
description: Test conventions
paths: ["**/*.test.ts"]
---
Vitest only · `describe` per public symbol · no shared mutable fixtures.
```

### Anti-patterns

- Monolithic CLAUDE.md with all conventions under headers — model must infer which section applies, frequently gets it wrong ❌
- Per-directory CLAUDE.md files for conventions that span many directories — requires duplication in every directory ❌
- Skills for automatic convention loading — skills require manual invocation, not path-triggered loading ❌
- Rule file without `paths:` — effectively another root CLAUDE.md loaded on every turn ❌

### Problem: Cross-directory conventions applied inconsistently

**Scope:** `claude-code`

**Problem statement:** Test files spread throughout the codebase (`Button.test.tsx` next to `Button.tsx`) receive inconsistent convention enforcement — Claude frequently applies component conventions to test files.

**Root cause:** Monolithic CLAUDE.md with conventions under separate headers requires implicit inference about which section applies. This is unreliable.

**Key decision:** `.claude/rules/` with glob patterns vs monolithic CLAUDE.md vs per-directory CLAUDE.md vs skills.

**Solution:** Create `.claude/rules/testing.md` with `paths: ["**/*.test.tsx", "**/*.spec.ts"]`. Rules load automatically when editing matching files, regardless of directory location.

### Use case: Rule File Design

A monorepo has:

- React components under `src/components/**`
- Express API handlers under `src/api/**`
- Database models under `src/db/**`
- Test files scattered throughout as `*.test.ts` alongside source files

Design the `.claude/rules/` file structure. Write the YAML frontmatter for each rule file and specify which conventions each would contain. (See rule files in *How to* above for the complete answer.)
