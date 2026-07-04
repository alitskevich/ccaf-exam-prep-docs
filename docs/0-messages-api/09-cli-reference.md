# CLI Reference

A focused reference of `claude` commands and flags called out in this book.

For the full upstream list — including flags shipped after this revision — see [code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference).

`claude --help` does not list every flag; absence from `--help` does not mean a flag is missing.

## Commands

| Command | What it does |
|---|---|
| `claude` | Start an interactive session in the current directory. |
| `claude "query"` | Start interactive with an initial prompt. |
| `claude -p "query"` | Non-interactive (print) mode — runs once and exits. The required form for CI and scripted calls. |
| `claude -c` | Resume the most recent session in this directory. |
| `claude -r <id\|name>` | Resume a specific session by ID or name. Requires the same `cwd`. |
| `claude update` | Update to the latest released version. |
| `claude install [version\|stable\|latest]` | Install or pin a specific binary version. |
| `claude auth login` / `logout` / `status` | Manage Anthropic authentication. `--console` for API-key billing, `--sso` to force SSO. |
| `claude agents` | Open the agent view; lists subagents when output is piped. |
| `claude mcp` | Configure MCP servers (see the MCP docs). |
| `claude plugin <sub>` | Manage plugins: `install`, `list`, `remove`. Alias: `claude plugins`. |
| `claude setup-token` | Generate a long-lived OAuth token for CI. |
| `claude project purge [path]` | Delete local state (transcripts, task lists, history) for a project. `--dry-run` to preview. |
| `claude attach <id>` / `logs <id>` / `stop <id>` / `rm <id>` / `respawn <id>` | Manage background sessions. |
| `claude remote-control` | Start a Remote Control server to drive Claude Code from claude.ai or the Claude app. |
| `claude ultrareview [target]` | Run ultrareview non-interactively. `--json` for raw payload, `--timeout <min>` to override the 30-min default. |

## Session control

- [ ] `--print` / `-p` : Non-interactive mode. Without it Claude waits on a TTY and the job hangs until timeout.
- [ ] `--continue` / `-c` : Load the most recent conversation in this directory.
- [ ] `--resume` / `-r` : Continue a previous session by ID or name. Requires the same `cwd`.
- [ ] `--fork-session` : Branch a session into an independent copy from a shared base (use with `--resume` or `--continue`). For A/B comparisons.
- [ ] `--session-id <uuid>` : Use a specific UUID for the conversation, e.g. to correlate with external systems.
- [ ] `--name` / `-n` : Display name for the session; resumable with `claude --resume <name>`.
- [ ] `--worktree` / `-w` : Run inside a git worktree at `<repo>/.claude/worktrees/<name>`. Enables parallel sessions on different features. Pass `#<n>` or a PR URL to branch from a pull request.
- [ ] `--tmux` : Create a tmux session for the worktree (requires `--worktree`). Uses iTerm2 native panes when available.
- [ ] `--no-session-persistence` : Don't save the session to disk (print mode only). Same effect as `CLAUDE_CODE_SKIP_PROMPT_HISTORY`.
- [ ] `--from-pr` : Resume sessions linked to a specific PR (number or GitHub/GitLab/Bitbucket URL).

## Output & input formats

- [ ] `--output-format` : Print-mode output shape: `text`, `json`, `stream-json`.
- [ ] `--input-format` : Print-mode input shape: `text`, `stream-json`.
- [ ] `--json-schema` : Schema-validated structured output (print mode). Pair with `--permission-mode dontAsk` for CI gating.
- [ ] `--include-partial-messages` : Emit partial streaming events. Requires `--print` and `--output-format stream-json`.
- [ ] `--include-hook-events` : Include all hook lifecycle events in the output stream. Requires `--output-format stream-json`.
- [ ] `--replay-user-messages` : Re-emit stdin user messages on stdout for acknowledgment. Requires `--input-format stream-json` and `--output-format stream-json`.

## Permissions & safety

- [ ] `--permission-mode` : Start in `default` / `acceptEdits` / `plan` / `auto` / `dontAsk` / `bypassPermissions`. Overrides `defaultMode` from settings.
- [ ] `--allowedTools` : Tools allowed without prompting (pattern matching applies, e.g. `"Bash(git log *)"`).
- [ ] `--disallowedTools` : Tools removed from the model's context entirely.
- [ ] `--tools` : Restrict which built-in tools are loaded. `""` disables all, `"default"` keeps all, otherwise a comma list.
- [ ] `--dangerously-skip-permissions` : Skip permission prompts — equivalent to `--permission-mode bypassPermissions`. Use only in isolated VMs/containers.
- [ ] `--allow-dangerously-skip-permissions` : Add `bypassPermissions` to the `Shift+Tab` mode cycle without starting in it.
- [ ] `--permission-prompt-tool` : MCP tool that handles permission prompts in non-interactive mode.

## Model & system prompt

- [ ] `--model` : Set the model, e.g. `sonnet`, `opus`, or a full ID like `claude-sonnet-4-6`. Overrides the `model` setting and `ANTHROPIC_MODEL`.
- [ ] `--fallback-model` : Fallback model used when the primary is overloaded (print mode only).
- [ ] `--effort` : Effort level: `low` / `medium` / `high` / `xhigh` / `max` (model-dependent). Does not persist.
- [ ] `--system-prompt` : Replace the entire default system prompt with the given text.
- [ ] `--system-prompt-file` : Replace with the contents of a file. Mutually exclusive with `--system-prompt`.
- [ ] `--append-system-prompt` : Append text to the default prompt; preserves tool guidance and safety instructions.
- [ ] `--append-system-prompt-file` : Append a file's contents to the default prompt.
- [ ] `--exclude-dynamic-system-prompt-sections` : Move per-machine sections (cwd, env, memory paths, git flag) out of the system prompt and into the first user message — improves cross-machine prompt-cache reuse. Only applies with the default system prompt.

## Working tree & IDE

- [ ] `--add-dir` : Additional directories Claude may read or edit. Most `.claude/` config is *not* discovered from these paths.
- [ ] `--ide` : Connect to a running IDE on startup if exactly one valid IDE is detected.
- [ ] `--chrome` / `--no-chrome` : Enable or disable Chrome browser integration.

## Extensions

- [ ] `--mcp-config` : Load MCP servers from JSON file(s) or strings (space-separated).
- [ ] `--strict-mcp-config` : Use *only* the MCP servers from `--mcp-config`; ignore all other MCP configurations.
- [ ] `--agents` : Define custom subagents inline as JSON.
- [ ] `--agent` : Use a specific subagent for the current session.
- [ ] `--plugin-dir` : Load a plugin from a directory or `.zip` for this session. Repeat the flag for multiple plugins.
- [ ] `--plugin-url` : Fetch and load a plugin `.zip` from a URL for this session.
- [ ] `--disable-slash-commands` : Disable all skills and commands for this session.
- [ ] `--settings` : Inline JSON or path to a settings file, overriding matching keys in `settings.json` for this session.
- [ ] `--setting-sources` : Comma list of setting layers to load: `user`, `project`, `local`.
- [ ] `--bare` : Skip auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and `CLAUDE.md` for faster scripted starts. Sets `CLAUDE_CODE_SIMPLE`.

## Budget & loop control

- [ ] `--max-turns` : Cap agentic turns (print mode). Exits with an error when the limit is reached. No limit by default.
- [ ] `--max-budget-usd` : Stop after spending this many dollars on API calls (print mode).

## Background & remote sessions

- [ ] `--bg` : Launch as a background agent and return immediately. Combine with `--agent` to run a specific subagent.
- [ ] `--remote` : Create a new web session on claude.ai with the given task description.
- [ ] `--remote-control` / `--rc` : Interactive session also controllable from claude.ai or the Claude app.
- [ ] `--teleport` : Resume a web session in your local terminal.
- [ ] `--teammate-mode` : Agent-team display mode: `auto`, `in-process`, or `tmux`.
- [ ] `--channels` : (Research preview) MCP servers whose channel notifications Claude listens for this session.

## Diagnostics

- [ ] `--verbose` : Show full turn-by-turn output. Overrides the `viewMode` setting for this session.
- [ ] `--debug` : Debug logging with optional category filter (`"api,hooks"`, `"!statsig,!file"`).
- [ ] `--debug-file <path>` : Write debug logs to a specific path; implies `--debug`. Takes precedence over `CLAUDE_CODE_DEBUG_LOGS_DIR`.
- [ ] `--init` / `--init-only` : Run Setup (and `SessionStart`) hooks before the session; `--init-only` exits afterwards.
- [ ] `--maintenance` : Run Setup hooks with the `maintenance` matcher before the session (print mode only).
- [ ] `--betas` : Beta headers to include in API requests (API-key users only).
- [ ] `--version` / `-v` : Print the version.
