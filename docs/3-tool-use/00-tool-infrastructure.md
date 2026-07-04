# Tool Infrastructure

For >30 tools, use **tool search** instead of sending every schema. For third-party integrations, use **MCP connector**. For tight tool-chain loops, use **programmatic tool calling** to avoid round-trip waste.

### Key Concepts

| If you need... | Reach for | Why |
|---|---|---|
| Reusable capability bundles | **Agent Skills** | Progressive disclosure |
| Third-party integrations fast | **MCP connector** | No custom tool plumbing |
| Hundreds+ of tools | **Tool search** | Avoid context blowout |
| Tight tool-chain loops | **Programmatic tool calling** | One round trip, many calls |
| Faster perceived latency on big tool inputs | **Fine-grained streaming** | Stream params live |

- **Tool search modes** (`ENABLE_TOOL_SEARCH`): unset/`true` always-on (default), `auto` triggers when tool definitions exceed 10% of context, `auto:N` with custom percentage threshold, `false` disabled
- **Tool search limits**: 10,000 tools max; requires Sonnet 4 / Opus 4 or later (no Haiku); adds one extra round-trip on first discovery
- **Discovery optimization**: specific names like `search_slack_messages` outperform `query_slack`; add a system prompt section listing tool categories to help the model query
- **MCP tool name format**: `mcp__<server-name>__<tool-name>` — wildcard `mcp__github__*` allows all tools from a server. Use this in `allowedTools`
- **MCP server connection timeout** defaults to 60 seconds; SDK doesn't run OAuth flows — complete the flow in your app and pass the token via `headers: { Authorization: "Bearer ..." }`
- **MCP transport types** (SDK): `stdio` (local process), `http` / `sse` (remote), in-process via `createSdkMcpServer`
- MCP connector latency = your MCP server's latency; cache server-side aggressively

### How to

```python
# Hundreds of tools — let Claude search instead of sending all schemas
client.beta.messages.create(
    model="claude-opus-4-7",
    tool_search={"enabled": True},
    mcp_servers=[{"url": "https://mcp.linear.app", "auth": {...}}],
    tools=[*core_tools, *deferred_tools],   # deferred have name only
    extra_headers={"anthropic-beta": "tool-search-2025-11-15"},
)
```

- **Send schemas upfront for ≤30 tools; enable tool search above that.** Tool search adds a discovery turn — only worth it when context cost would dominate
- **Use the MCP connector for community-maintained integrations.** Build a custom MCP server only when none exists
- **Cap programmatic tool calling iterations server-side.** A tight verify-fix-verify loop can run *many* tools per turn; runaway cost shows up on the bill, not in your client
- **Stream large tool inputs** with fine-grained streaming when the input itself is the latency tail (long prompts, big payloads)
- **Keep skill descriptions one line** — they're always loaded, defeating progressive disclosure when verbose

### Anti-patterns

- Sending all tool schemas upfront when you have hundreds — wastes context
- Building custom MCP servers for standard integrations when community servers exist
- Enabling programmatic tool calling without a server-side iteration cap — runaway cost risk
- Bloated skill descriptions — they're always loaded, defeating progressive disclosure
