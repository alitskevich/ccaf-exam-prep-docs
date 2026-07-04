# MCP Configuration

Use `.mcp.json` for team-shared servers (with `${ENV_VAR}` references for secrets) and `~/.claude.json` for personal/experimental servers — never commit literal tokens.

### Key Concepts

- `.mcp.json` is checked in — use `${ENV_VAR}` expansion, never literal tokens
- `~/.claude.json` overrides project config silently — check both layers when debugging tool behavior
- A teammate without a referenced env var sees confusing "MCP server failed" errors — document required env vars in the README
- Tool description quality, not capability, drives selection — minimal MCP descriptions cause the agent to prefer better-described built-ins
- **Three transport types** (SDK): `stdio` (local process via `command` + `args`), `http` / `sse` (remote via `type` + `url` + `headers`), in-process via `createSdkMcpServer`
- **OAuth flow is your responsibility** — the SDK doesn't run OAuth flows. Complete the flow in your app, then pass the access token via `headers: { Authorization: "Bearer ..." }`
- **MCP server connection timeout** defaults to 60 seconds
- **Don't use `acceptEdits` to approve MCP** — it only auto-approves file edits. Prefer wildcard `allowedTools`: `["mcp__github__*", "mcp__db__query"]`
- Tools from all configured MCP servers are **discovered at connection time** and available simultaneously — combine with **tool search** when servers expose many tools

### How to

- Use `.mcp.json` for team-shared servers; reference secrets via `${ENV_VAR}`
- Use `~/.claude.json` for personal, experimental, or local-prototype servers
- Write rich tool descriptions for MCP servers — purpose, inputs, outputs, when to prefer over alternatives

```jsonc
// .mcp.json — checked into the repo
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "jira": {
      "type": "http",
      "url": "https://mcp.internal.company.com",
      "headers": { "Authorization": "Bearer ${JIRA_API_KEY}" }
    }
  }
}
```

```jsonc
// ~/.claude.json — personal, not shared
{
  "mcpServers": {
    "experimental-rag": {
      "command": "node",
      "args": ["/Users/me/proto/rag-server/dist/index.js"]
    }
  }
}
```

### Anti-patterns

- Committing API tokens directly in `.mcp.json` ❌
- Putting team-shared MCP servers in `~/.claude.json` — invisible to teammates ❌
- Writing minimal MCP tool descriptions — agent prefers built-in tools with better descriptions ❌
- Building custom MCP servers for standard integrations when community servers exist ❌

### Problem: Agent ignores capable MCP tool, uses inferior built-in

**Scope:** `mcp`

**Problem statement:** An MCP tool provides advanced code analysis capabilities, but the agent consistently uses the built-in Grep tool instead, missing the MCP tool's richer output.

**Root cause:** The agent selects tools based on descriptions, and the built-in Grep tool's description better communicates what it does.

**Solution:** Expand the MCP tool's description to explain its capabilities, input format, output format, and when to prefer it over built-in tools.

**Anti-pattern:** Assuming tool capability alone drives selection — description quality is the primary selection mechanism.

### Use case: MCP Configuration

Write a `.mcp.json` for a team's shared environment with a GitHub MCP server using `GITHUB_TOKEN` and an internal Jira server at `https://mcp.internal.company.com` using `JIRA_API_KEY`. Then describe what you would put in `~/.claude.json` for a personal experimental server that should not affect teammates. (See *How to* above for the configuration.)
