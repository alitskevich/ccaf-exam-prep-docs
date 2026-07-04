# CLAUDE.md Hierarchy

Place each rule at the level where it applies — universal vs path-scoped vs personal vs org-policy — so it loads on the right turns and stays visible to the right people.

### Key Concepts

| If it's... | Put it in |
|---|---|
| A rule that should apply always | `CLAUDE.md` |
| A rule for one path/area | `.claude/rules/<name>.md` with `paths:` |
| A personal preference, all projects | `~/.claude/CLAUDE.md` |
| A personal note, this project only | `CLAUDE.local.md` |
| An observation from a past conversation | Auto memory (Claude writes it) |
| An org-wide policy | Managed `CLAUDE.md` |

- Path-scoped rules in `.claude/rules/` only load when matching files are touched, keeping the per-turn context lean
- **Directory-level CLAUDE.md** (e.g., `packages/api/CLAUDE.md`) loads only when working inside that subtree — useful for monorepos where each package has distinct conventions
- **Modular includes via `@path` syntax:** keep the root `CLAUDE.md` short by referencing topic files (e.g., `@./standards/testing.md`); use `@` immediately before the path (no space); relative paths resolve relative to the file containing the import; **maximum import nesting depth is 5**
- **Auto memory** at `~/.claude/projects/<project>/memory/`: `MEMORY.md` is the index — first 200 lines / 25KB load every session; topic files (`debugging.md`, `user_role.md`) load on demand. Persists until pruned; review periodically
- **Toggle auto memory** with `/memory` command, `autoMemoryEnabled` in `settings.json`, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` env var
- **Auto memory saves**: user profile, feedback (corrections + validated approaches), project state, external references. **Does NOT save**: code patterns derivable from the codebase, git history, debugging recipes, anything in CLAUDE.md
- **`/memory` command** opens loaded instruction files for review — useful when debugging inconsistent behavior to verify which files are loaded
- **Layer rules at the level where they apply.** Load order: Managed (org-wide) → User (`~/.claude/`) → Project (`./.claude/`) → Local (`.local`). Layers merge; later layers override earlier — except managed `deny` rules, which always win

```markdown
<!-- project/CLAUDE.md — modular via @path includes -->
# Acme conventions

@./standards/coding-style.md
@./standards/testing.md
@./standards/api.md
```

### How to

```
repo/
├── CLAUDE.md                       # checked-in, all-developer rules
├── CLAUDE.local.md                 # gitignored, personal
└── .claude/
    └── rules/
        ├── api.md       (paths: src/api/**)
        ├── tests.md     (paths: **/*.test.ts)
        └── db.md        (paths: src/db/**)

~/.claude/CLAUDE.md                 # cross-project preferences
/etc/claude-code/CLAUDE.md          # managed org policy (read-only to dev)
```

- **Keep `CLAUDE.md` under 200 lines.** Test for each line: *"If I removed this, would Claude make a mistake?"* If no — prune. Bloated `CLAUDE.md` taxes every session forever
- **Move workflows to skills, path-rules to `.claude/rules/`.** Anything Claude can derive from the code itself doesn't belong in `CLAUDE.md`
- **Verify `CLAUDE.local.md` is in `.gitignore`** — the most common slip is personal preferences leaking into PRs

### Anti-patterns

- Putting team conventions in `~/.claude/CLAUDE.md` — invisible to teammates, never ships via version control ❌
- Monolithic CLAUDE.md with all conventions — hard to maintain, loads irrelevant context ❌
- Placing slash command definitions in CLAUDE.md body — commands belong in `.claude/commands/` ❌
- A rule file in `.claude/rules/` without `paths:` — effectively another root `CLAUDE.md` ❌
- Putting API rules in the root `CLAUDE.md` loads them on every turn even when editing CSS — wastes context ❌
- Personal prefs in `CLAUDE.md` (instead of `CLAUDE.local.md`) leak into PRs and confuse teammates ❌

### Problem: New developer ignores team conventions

**Scope:** `claude-code`

**Problem statement:** A new team member's Claude Code session does not follow team coding conventions, despite the rest of the team applying them consistently.

**Root cause:** Conventions are in `~/.claude/CLAUDE.md` (user level) — not checked into version control, invisible to new team members.

**Key decision:** User-level CLAUDE.md vs project-level CLAUDE.md vs directory-level CLAUDE.md vs shared via Slack.

**Solution:** Move team conventions to project-level `CLAUDE.md` so they are shared via version control and applied to all team members automatically.

### Use case: CLAUDE.md Configuration

A monorepo has these requirements:

- Universal coding standards for all developers
- API-specific naming conventions (only for `src/api/**`)
- Test file conventions for `*.test.ts` files spread throughout the repo
- Personal formatting preferences (one developer only)

Design the CLAUDE.md hierarchy. Which level and mechanism handles each requirement?

```
repo/
├── CLAUDE.md                          # Universal standards (all devs)
├── CLAUDE.local.md                    # Personal prefs (gitignored)
└── .claude/rules/
    ├── api-naming.md      # paths: src/api/**
    └── test-conventions.md # paths: **/*.test.ts
```

```markdown
<!-- .claude/rules/api-naming.md -->
---
name: api-naming
description: REST naming conventions
paths: ["src/api/**"]
---
- Routes use kebab-case: `/api/user-profile` not `/api/userProfile`.
- Handlers exported as `handle<Resource><Verb>` e.g. `handleUserCreate`.
```
