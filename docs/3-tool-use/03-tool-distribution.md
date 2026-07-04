# Tool Distribution Across Agents

Assign each agent only the tools needed for its role — over-provisioning degrades selection reliability and causes role drift.

### Key Concepts

- 18 tools per agent degrades selection accuracy; target **4–5 tools per agent**
- Tools outside an agent's specialization cause role drift — the agent optimizes for what it can do, not what it should do
- Scoped cross-role tools handle high-frequency cross-boundary needs (e.g., `verify_fact` for synthesis); complex cases route through the coordinator
- `tool_choice` options: `"auto"` (may use tool or text), `"any"` (must call a tool), forced `{"type": "tool", "name": "..."}` (must call specific tool)
- Shared tools can repeat across agents — `verify_fact` ending up everywhere is fine

### How to

- Restrict each subagent's tool set to those relevant to its role
- Replace generic tools with constrained alternatives (e.g., `fetch_url` → `load_document` that validates document URLs only)
- Provide scoped cross-role tools for high-frequency needs rather than full tool access
- Use `tool_choice: "any"` to guarantee the model calls a tool instead of returning conversational text
- Use forced `tool_choice` to ensure a specific tool is called first (e.g., `extract_metadata` before enrichment), then process subsequent steps in follow-up turns

### Anti-patterns

- Giving all tools to all agents "for flexibility" — increases misselection and role drift
- Giving a synthesis agent full web search access — agent conducts research instead of synthesizing
- End-of-pass batched verification — blocks synthesis steps that depend on earlier verified facts
- Proactive caching by the search agent — cannot predict what synthesis will need
- Coordinator with too many tools starts *doing* work instead of delegating

### Problem: Synthesis agent over-provisioned with all search tools

**Scope:** `agent-sdk`

**Problem statement:** To eliminate coordinator round-trips during claim verification, the synthesis agent is given access to all web search tools. It conducts additional research rather than synthesizing, producing inconsistent outputs and blurring the role boundary.

**Key decision:** Scoped `verify_fact` tool vs all search tools vs end-of-pass batching vs proactive caching.

**Solution:** Give the synthesis agent a scoped `verify_fact` tool covering the 85% simple case (date, name, statistic lookups). Complex verifications (15%) route through the coordinator to the web search agent.

### Use case: Tool Distribution Design

A multi-agent research system has: coordinator, web search agent, document analysis agent, synthesis agent.

Available tools (16 total):

- **Web:** `search_web`, `fetch_url`, `extract_links`, `parse_search_results`
- **Documents:** `read_pdf`, `extract_tables`, `extract_citations`, `summarise_document`
- **Synthesis:** `synthesise_findings`, `identify_contradictions`, `generate_outline`, `format_report`
- **Shared:** `verify_fact`, `normalise_date`, `resolve_citation`, `check_recency`

Assign tools to agents. No agent should have more than 6 tools. Justify each exclusion.

```python
AGENT_TOOLS = {
  "coordinator":     ["delegate", "verify_fact", "normalise_date"],          # 3 — orchestrates only
  "web_search":      ["search_web", "fetch_url", "extract_links",
                      "parse_search_results", "check_recency", "normalise_date"],   # 6
  "document":        ["read_pdf", "extract_tables", "extract_citations",
                      "summarise_document", "resolve_citation", "verify_fact"],     # 6
  "synthesis":       ["synthesise_findings", "identify_contradictions",
                      "generate_outline", "format_report",
                      "verify_fact", "resolve_citation"],                            # 6
}
```
