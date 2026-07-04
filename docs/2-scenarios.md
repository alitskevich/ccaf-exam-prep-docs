# Exam Scenarios

Each scenario frames a set of questions in a realistic production context. The exam picks **4 of the 8 scenarios** below at random.

### Scenario 1: Customer Support Agent

An agent handles returns, billing disputes, and account issues via the Claude Agent SDK with MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Target: 80%+ first-contact resolution with appropriate escalation.

```ts
// Claude Agent SDK — agent loop driven by stop_reason, not text parsing
import { Agent } from "@anthropic-ai/claude-agent-sdk";

const agent = new Agent({
  name: "support-agent",
  systemPrompt: `Resolve returns / billing / account. Escalate when:
    - refund > $500 (policy gap)
    - customer explicitly asks for a human
    - two consecutive tool errors without recovery`,
  mcpServers: { crm: { command: "node", args: ["./mcp-crm.js"] } },
  allowedTools: ["get_customer", "lookup_order", "process_refund", "escalate_to_human"],
});

// Hook enforces business rule programmatically — prompts cannot guarantee compliance
agent.on("PostToolUse", (call) => {
  if (call.tool === "process_refund" && call.input.amount > 500) {
    return { block: true, reason: "Exceeds auto-refund ceiling — must escalate" };
  }
});

while (true) {
  const turn = await agent.step(input);
  if (turn.stop_reason === "end_turn") break;
  if (turn.stop_reason === "tool_use") input = await runTools(turn.tool_calls);
}
```

### Scenario 2: Code Generation with Claude Code

Claude Code accelerates generation, refactoring, debugging, and documentation. Integration covers custom slash commands, CLAUDE.md configuration, and planning-mode selection.

```md
<!-- .claude/CLAUDE.md  — project-level, visible to the whole team
     (NOT ~/.claude/CLAUDE.md, which is user-only) -->
- 2-space indent, double quotes, no semicolons
- After editing libs/*, run `npm test`
- Public API changes require an entry in CHANGELOG.md

@docs/architecture.md   <!-- @import pulls in larger reference docs on demand -->
```

```md
<!-- .claude/rules/migrations.md  — path-scoped via frontmatter glob -->
---
globs: ["db/migrations/**/*.sql"]
---
Never DROP without a paired backfill commit. Every migration is reversible.
```

```md
<!-- .claude/commands/triage.md  — team slash command -->
---
allowed-tools: [Read, Grep, Glob]
argument-hint: <bug-id>
context: fork        # isolates from main session context
---
Triage bug $ARGUMENTS — read related files, summarize root cause in <200 words.
```

```text
# Mode selection (judgment, not a flag):
#   plan mode      → architectural changes, 3+ files, public API touches
#   direct execute → single-file edits, mechanical refactors
#   /compact       → token budget approaching limit
#   /memory        → update CLAUDE.md from inside the session
#   --resume       → continue a previous session by name
```

### Scenario 3: Multi-Agent Research System

A coordinator delegates to specialized subagents (web research, document analysis, synthesis) and produces complete reports with citations.

```ts
// Coordinator — emits ALL Task calls in a SINGLE response for true parallelism
// (multiple Task calls across separate turns is the anti-pattern)
const coordinator = new Agent({
  systemPrompt: "Decompose query → dispatch subagents in parallel → synthesize cited report.",
  allowedTools: ["Task"],
});

await coordinator.run({
  toolCalls: [
    { name: "Task", input: { subagent: "web-research",  prompt: q, context: { sources } } },
    { name: "Task", input: { subagent: "doc-analysis",  prompt: q, context: { docIds } } },
  ],   // ← parallel dispatch, one response
});

// Subagent definitions — narrow allowedTools (4–5 each), explicit context passing.
// Subagents do NOT inherit parent context — pass it in the prompt.
const webResearch = new AgentDefinition({
  allowedTools: ["web_search", "fetch_url"],
  // Structured error so coordinator can recover — never empty success, never generic message
  onError: (e) => ({
    isError: true, category: "transient", retryable: true,
    partial: e.partialResults ?? [], message: e.message,
  }),
});

// Synthesis runs as an INDEPENDENT instance — same-session self-review is biased
const synthesis = new AgentDefinition({
  allowedTools: ["Read"],
  systemPrompt: "Cite every claim with {source_id, quote}. Annotate conflicts and coverage gaps.",
});
```

### Scenario 4: Developer Productivity Tools

The agent helps engineers explore unfamiliar codebases, generate boilerplate, and automate routine tasks via built-in tools (Read, Write, Bash, Grep, Glob) plus MCP servers.

```ts
// Delegate broad discovery to the Explore subagent — keeps main context lean
await main.run({
  toolCalls: [
    { name: "Task", input: { subagent: "Explore", prompt: "find auth middleware usage" } },
  ],
});

// Built-in tool selection (the exam tests purpose, not syntax):
//   Glob   → file enumeration by pattern  ("src/**/*.ts")
//   Grep   → symbol / keyword across known scope
//   Read   → known path, content matters
//   Edit   → diff-only modification of an existing file
//   Write  → new files or full rewrites only
//   Bash   → shell-only ops (tests, builds, git)
```

```jsonc
// .mcp.json — env var expansion + project scope (vs ~/.mcp.json user scope)
{
  "mcpServers": {
    "jira":   { "command": "npx", "args": ["@org/mcp-jira"],
                "env": { "JIRA_TOKEN": "${JIRA_TOKEN}" } },
    "github": { "command": "npx", "args": ["@org/mcp-github"],
                "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" } }
  }
}
```

### Scenario 5: Claude Code for Continuous Integration

Claude Code runs in CI/CD for automated reviews, test generation, and PR feedback. Prompts must minimize false positives.

```yaml
# .github/workflows/review.yml — non-interactive Claude Code via -p / --print
# (NOT a fictional CLAUDE_HEADLESS env var, NOT a --batch flag)
- name: Claude review
  run: |
    git diff origin/main...HEAD | \
      claude -p "Review the diff for bugs and security issues. Be conservative — \
        only flag findings you can defend with file:line + reproduction steps. \
        Follow the few-shot examples below for severity calibration." \
        --output-format json \
        --json-schema ./schemas/review.json \
        > review.json
```

```ts
// False-positive reduction strategy:
//   1. Few-shot examples > extra instruction paragraphs (format consistency)
//   2. TWO independent passes — never consensus-vote (suppresses real bugs)
//   3. Sync API for blocking pre-merge checks
//   4. Batch API only for nightly / overnight reports (no latency SLA)
const reviewPrompt = `
<example issue="false-positive">
  Code: const x = users.filter(u => u.active)
  Verdict: NOT a bug — .filter is non-mutating.
</example>
<example issue="real-bug">
  Code: db.exec("DELETE FROM x WHERE id=" + id)
  Verdict: SQL injection — file:line:column + safe alternative.
</example>`;
```

### Scenario 6: Structured Data Extraction

The system extracts information from unstructured documents, validates output with JSON schemas, and handles edge cases.

```ts
// tool_use with strict JSON Schema — forced tool_choice eliminates free-form drift
const extract = {
  name: "extract_invoice",
  input_schema: {
    type: "object",
    required: ["invoice_id", "total"],
    properties: {
      invoice_id: { type: "string" },
      total:      { type: "number" },
      vendor:     { type: ["string", "null"] },              // nullable, NOT omit — prevents hallucination
      classification:        { enum: ["bill", "credit", "other"] },
      classification_detail: { type: "string" },             // "other" + detail pattern
    },
  },
};

const resp = await client.messages.create({
  model: "claude-sonnet-4-6",
  tools: [extract],
  tool_choice: { type: "tool", name: "extract_invoice" },    // forced selection
  messages: [{ role: "user", content: docText }],
});
```

```python
# Pydantic — semantic validation + retry loop feeding errors back to Claude
class Invoice(BaseModel):
    invoice_id: str
    total: float
    vendor: str | None = None

    @validator("total")
    def positive(cls, v):
        assert v > 0, "total must be positive"
        return v

for attempt in range(3):
    try:
        return Invoice(**parsed)
    except ValidationError as e:
        parsed = ask_claude_to_fix(parsed, errors=e.errors())   # retry with feedback

# Stratified accuracy by document type + field — aggregate 97% can hide
# a 60% segment failure. Human review threshold per (doc_type, field).
```

```ts
// Batch API for overnight bulk extraction (50% cheaper, ≤24h SLA, custom_id correlation)
// Sync API for interactive / blocking flows.
```

### Scenario 7: Conversational AI Architecture Patterns

Multi-turn conversational systems: context-window management, instruction persistence, memory strategies, safe tool execution, and handling ambiguous or conflicting input.

```ts
// Persistent instructions live in the system prompt — NOT replayed each turn
const session = await sdk.createSession({
  system: "You are a tutor. Always cite the lesson section when explaining.",
  name:   "tutor-2026-05-09-alex",                     // named session for resume
});

// Context-window strategy:
//   /compact          → progressive summarization when token budget tightens
//   fork_session      → branch experimentally without polluting the main thread
//   scratchpad files  → extract structured facts from verbose tool output
//   position-aware    → put critical instructions at TOP and BOTTOM
//                       (counters "lost in the middle" attention dilution)
const branch = await session.fork({ from: lastTurnId });

await fs.writeFile("scratch/facts.json", extractFacts(verboseToolOutput));
// Future turns Read scratch/facts.json instead of re-fetching the raw output

// Ambiguity handling: ask ONE clarifying question rather than guess.
// Conflicting inputs: surface the conflict with both interpretations and a recommendation.
```

```ts
// Safe tool execution — the hook is the enforcement layer, not the prompt
agent.on("PostToolUse", (call) => {
  if (call.tool === "send_email" && call.input.to.endsWith("@external.com")) {
    return { block: true, reason: "External recipient — requires human approval" };
  }
});
```

### Scenario 8: Agentic AI Tools *(content missing — help us fill it in!)*

This scenario has been reported by candidates but is not yet covered. If you have seen questions from it, please share them in [GitHub Issues](https://github.com/paullarionov/claude-certified-architect/issues).

The likely focus is **MCP tool & resource design**:

```ts
// Splitting > consolidating — distinct descriptions drive accurate selection.
// Anti-pattern: one giant `manage_order(action, ...)` — ambiguous to the model.

// Good — narrow, well-described tools (4–5 per agent role)
const tools = [
  {
    name: "lookup_order",
    description: `Retrieve a single order by ID.
      USE WHEN: user provides an order number, or asks "where is order X".
      DO NOT USE FOR: customer-wide history (use list_orders_by_customer).`,
    input_schema: { /* ... */ },
  },
  {
    name: "cancel_order",
    description: `Cancel an unshipped order. Idempotent.
      USE WHEN: user requests cancellation AND order.status === "pending".
      DO NOT USE FOR: shipped orders (use start_return).`,
    input_schema: { /* ... */ },
  },
];

// Description quality > routing classifier. Improve descriptions FIRST;
// only reach for a classifier if disambiguation still fails after that.
```

```ts
// Structured error response — coordinator can recover intelligently
return {
  isError: true,
  category: "transient",            // transient | business | permission
  retryable: true,
  partial:   { order_id, status: "lookup_failed" },
  message:   "CRM 503 after 3 retries",
};
// Anti-patterns: empty success, generic "search unavailable", swallowed exceptions.
```

```ts
// MCP resources vs tools:
//   resources → static / catalog content the model READS  (docs, schemas, prompts)
//   tools     → ACTIONS with side effects or dynamic queries
// Choosing the wrong primitive is a common distractor.
```
