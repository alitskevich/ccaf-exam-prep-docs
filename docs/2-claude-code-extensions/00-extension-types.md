# Extension Types

Pick the extension that matches the *intended trigger*.

### Key Concepts

| If you want... | Reach for | Why |
|---|---|---|
| Lint after every edit | **Hook** (`PostToolUse`) | Deterministic, runs every time |
| A reusable `/deploy` workflow | **Skill** | Loaded on demand, body kept out of context until used |
| Project rules ("use 2-space indent") | **`CLAUDE.md`** | Always loaded |
| File-path-scoped rules | **`.claude/rules/*.md` with `paths:`** | Loaded only when relevant files are touched |
| Talk to Linear / Slack / a database | **MCP server** | Standard protocol, growing ecosystem |
| Heavy verbose investigation | **Subagent** | Isolated context, summary only returns |
| Competing approaches in parallel | **Agent team** | Independent sessions that message each other |
| Bundle the above for a team | **Plugin** | Distribute setup as one unit |
| Change Claude's voice/format | **Output style** | Wraps system prompt |
| Quick prompt shortcut | **Custom slash command** | Same mechanism as skills, no body |

- Subagents return a summary only — if you need raw outputs, write them to disk inside the subagent and read after
- Skills require **manual invocation** — they don't load on file paths (use `.claude/rules/` for that)
- **One project `.claude/` layout.** Keep all checked-in extensions discoverable in one place:

```text
  your-project/
  ├── CLAUDE.md
  ├── CLAUDE.local.md          # gitignored
  ├── .mcp.json
  └── .claude/
      ├── settings.json
      ├── rules/<name>.md
      ├── skills/<name>/SKILL.md
      ├── commands/<name>.md
      ├── output-styles/<name>.md
      └── agents/<name>.md
```

- **Same engine across surfaces.** Terminal, VS Code / JetBrains, desktop, web, CI (GitHub Actions / GitLab), Slack — all share `CLAUDE.md` + `.claude/settings.json`. Configure once
- **Pick by trigger semantics, not by name.** Must fire every time → hook. Model decides when → skill. Always-true facts → `CLAUDE.md`. Only when matching paths → `.claude/rules/`
- **Hooks must be fast (<2s) or async.** A slow hook blocks every tool call

### System prompt customization (Agent SDK)

| Method | Persistence | Customization |
|---|---|---|
| **CLAUDE.md** | Per-project file | Additions only |
| **Output styles** | Saved as files (`.md`) | Replace default |
| **`systemPrompt` with `append`** | Session only | Additions only |
| **Custom `systemPrompt` string** | Session only | Complete replacement |

The SDK's default system prompt is **minimal** — to get Claude Code's full prompt (coding guidelines, response style, environment context), use the preset:

```typescript
options: {
  systemPrompt: {
    type: "preset",
    preset: "claude_code",
    append: "Always include detailed docstrings and type hints in Python code."
  }
}
```

For cross-host cache reuse, set `excludeDynamicSections: true` to move per-session context (cwd, OS, date, git status) into the first user message instead of embedding it in the system prompt.

### Plugins

Plugins bundle skills, agents, hooks, and MCP servers. The SDK loads them from local paths only — for marketplace plugins, download first.

```typescript
options: {
  plugins: [
    { type: "local", path: "./my-plugin" },
    { type: "local", path: "/absolute/path/to/another-plugin" }
  ]
}
```

Plugin layout requires `.claude-plugin/plugin.json` manifest, and may include `skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`. Plugin skills are namespaced as `plugin-name:skill-name`.

### How to

- **Keep skill descriptions one line.** Bodies stay out of context until invoked, but descriptions are always loaded

### Anti-patterns

- Creating team commands in `~/.claude/commands/` — invisible to teammates, requires manual duplication ❌
- Embedding command definitions in CLAUDE.md body — creates text instructions, not an invocable command ❌
- Skills that write extensively to main session context without `context: fork` — verbose output pollutes coordination context ❌
- Using a `.claude/config.json` `commands` array — this mechanism does not exist ❌
- Picking a skill when you need a hook — skills require invocation; if the rule must always fire, you need a hook ❌
