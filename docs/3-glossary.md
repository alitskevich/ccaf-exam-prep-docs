# Glossary

A single lookup for the vocabulary used across this book — Messages API parameters, Agent SDK constructs, configuration and extensions, named patterns and effects, engineering techniques, and the technical English the prose leans on without defining. Definitions are scoped to how the book uses each term; reach for the corresponding chapter when more context is needed. For `claude` CLI commands and flags, see [CLI Reference](6-cli-reference.md).

## Configuration & Extensions

### Permission modes

- [ ] `acceptEdits` : Permission mode that auto-approves reads, file edits, and filesystem commands (`mkdir`, `mv`, `cp`, `rm`, `sed`, `touch`). Does not auto-approve MCP tools.
- [ ] `auto` : Mode that auto-approves everything with a *classifier model* blocking risky actions. Max/Team/Enterprise/API only; TypeScript SDK only.
- [ ] `bypassPermissions` : Mode with no permission checks — for isolated containers or VMs only. Ignores `allowedTools`.
- [ ] `dontAsk` : Permission mode for locked-down CI: only pre-approved tools run, everything else is denied without prompting.
- [ ] `plan` (mode) : Permission mode that allows reads only; produces a plan without edits.

### Project instructions & memory

- [ ] `CLAUDE.md` : Project-level instructions file checked into the repo. Always loaded into context.
- [ ] `CLAUDE.local.md` : Per-developer project instructions — gitignored, never shipped via version control.
- [ ] `@path` import : `CLAUDE.md` syntax pulling in a referenced file. Max nesting depth: 5.
- [ ] `paths:` : YAML frontmatter glob on a `.claude/rules/*.md` file. Loads the rule only when matching tool inputs occur.

### Skills

- [ ] `SKILL.md` : Skill definition file under `.claude/skills/<name>/`. Frontmatter: `name`, `description`, `context: fork`, `allowed-tools`, `argument-hint`.
- [ ] `context: fork` : `SKILL.md` frontmatter that runs the skill in an isolated subagent so its verbose output stays out of main session context.
- [ ] `argument-hint` : Frontmatter field on a `SKILL.md` or command file that prompts the developer for a required parameter.
- [ ] `$1` / `$2` / `$ARGUMENTS` : Positional argument substitutions inside a `.claude/commands/<name>.md` body.
- [ ] `` ! `cmd` `` : Custom-command syntax that executes a shell command and embeds its output into the prompt.

### Output styles, plugins

- [ ] Output style : A `.claude/output-styles/<name>.md` file that wraps the system prompt to change Claude's voice or format. Saved configuration, not session-only.
- [ ] Plugin : A bundle of skills, agents, hooks, and MCP servers distributed as one unit. Requires `.claude-plugin/plugin.json` manifest; plugin skills are namespaced `plugin-name:skill-name`.

### MCP

- [ ] `.mcp.json` / `~/.claude.json` : Project-level (committed) and user-level MCP server configs. User file silently overrides project file.
- [ ] `mcpServers` : Top-level key in `.mcp.json` mapping server names to `command`/`args`/`env` or `type`/`url`/`headers`.
- [ ] `mcp__<server>__<tool>` : MCP tool name format used in `allowedTools` matching. Wildcard `mcp__github__*` allows all tools from a server.
- [ ] `stdio` / `http` / `sse` / `createSdkMcpServer` : MCP transport types — local process, remote, server-sent events, in-process.
- [ ] Tool (MCP) : An action with side effects or an expensive fetch (e.g., `fetch_doc`, `create_jira_ticket`). Annotate destructive ones with `destructiveHint`; return `isError: true` instead of raising.
- [ ] Resource (MCP) : A passive catalog the model reads to decide (e.g., `docs://index`, `db://schema`, `jira://projects`). Promote a tool to a resource when it returns a static catalog.
- [ ] Prompt (MCP) : A reusable workflow template with arguments (e.g., `pr_review(pr_number)`). Expands into messages — does not execute. Centralizes prompt logic that would otherwise be duplicated across projects.
- [ ] `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` : MCP tool annotations. `readOnlyHint: true` enables parallel execution; others are informational.

### Hooks

- [ ] `UserPromptSubmit` : Hook event firing when the user prompt is sent.
- [ ] `PreToolUse` / `PostToolUse` : Hook events firing before / after a tool call. `PreToolUse` can block or modify; `PostToolUse` can normalize results.
- [ ] `PostToolUseFailure` / `PostToolBatch` : Hook events for tool failures and for batched tool completion (TS only).
- [ ] `PreCompact` : Hook event firing before context compaction runs.
- [ ] `Notification` : Hook event for status messages (`permission_prompt`, `idle_prompt`, `auth_success`, …).
- [ ] `Stop` : Hook event firing when the agent finishes.
- [ ] `SubagentStart` / `SubagentStop` : Hook events for subagent lifecycle.
- [ ] `SessionStart` / `SessionEnd` : TS-only hook events for session lifecycle. Python equivalent: shell hooks via `settings.json`.

### Caching & observability

- [ ] `ENABLE_PROMPT_CACHING_1H` : Env var enabling the 1-hour cache TTL — higher write cost, more cross-session hits.
- [ ] `TRACEPARENT` / `TRACESTATE` : OTel headers auto-injected into the SDK subprocess so the agent run nests under your application's trace.

## Concepts & Patterns

### Agentic loops & orchestration

- [ ] Agentic loop : Gather context → take action → verify → repeat. The single control variable is `stop_reason`.
- [ ] Coordinator–subagent pattern : One coordinator decomposes the task and routes work to specialized subagents. Subagents do not talk to each other.
- [ ] Dynamic adaptive decomposition : Decomposition pattern where subtasks are generated from intermediate results — for open-ended investigation.
- [ ] Explore subagent : Built-in subagent for verbose multi-step discovery. Main agent receives a summary only, preserving coordination context.
- [ ] Hub-and-spoke architecture : All inter-subagent communication routes through the coordinator. Subagents do not talk to each other and do not inherit parent context.
- [ ] Plan mode : Execution mode for complex / architectural / multi-file work. Investigation produces a plan that becomes the spec for direct execution.
- [ ] Plan-then-execute : Pattern: plan mode for investigation, then direct execution for implementation.
- [ ] Prompt chaining : Sequential decomposition where each step's verifiable output feeds the next. For predictable fixed-order workflows.
- [ ] Role drift : An over-provisioned agent optimizing for what it *can* do rather than what it *should*. The synthesis agent doing research instead of synthesizing is the canonical example.
- [ ] Worktree : A git working tree at a separate path. Enables parallel Claude Code sessions on different features without branch switching.
- [ ] Writer/Reviewer pattern : One session writes the change; a separate session reviews it. The writer's reasoning bias is excluded from the review.

### Context & memory

- [ ] Auto memory : Persistent notes under `~/.claude/projects/<project>/memory/`. `MEMORY.md` is the index; topic files load on demand.
- [ ] Codebase drift : Significant change in files since a prior session. Resuming on heavy drift makes the agent reason from a stale world; start fresh with a summary instead.
- [ ] Context bloat : Always-loaded context (CLAUDE.md, rules, system prompts) growing beyond what every session needs. Prune ruthlessly.
- [ ] Context degradation : In extended sessions, the model gives inconsistent answers, contradicts earlier findings, and references "typical patterns" instead of specific discovered details.
- [ ] Context killers : Reading large files (lockfiles, generated code, minified bundles), verbose command output, multi-file investigations in the main session.
- [ ] Lost-in-the-middle effect : Reduced attention to content in the middle of long inputs (notably past 20K tokens), even when fully present. Distinct from summarization loss — caused by position.
- [ ] Manifest-based crash recovery : A top-level `manifest.json` tracks each subagent's status; on resume the coordinator loads state files instead of replaying conversation history.
- [ ] Position-aware input layout : Place critical facts at the top AND bottom of long inputs; middle attention is least reliable.
- [ ] Progressive disclosure : Skill bodies stay out of context until invoked; only the one-line description is always loaded.
- [ ] Progressive summarization loss : Numeric IDs, amounts, dates, and percentages getting collapsed into "around X" by summarization. Pin transactional facts outside the compactable region.
- [ ] Speculative caching : Anti-pattern of proactively caching what an agent thinks downstream consumers will need. Cannot reliably predict; introduces stale data.

### Reasoning & decision strategy

- [ ] Adaptive thinking : Variable-budget reasoning mode that spends thinking tokens only when difficulty warrants. Configured via `thinking: {type: "adaptive", budget_tokens: N}`.
- [ ] Anti-pattern : A common-looking solution that fails for a specific named reason. The book uses `❌` to mark them.
- [ ] Distractor : An exam multiple-choice option a candidate with incomplete knowledge might choose. Categories: real technique in wrong context, over-engineered solution, non-existent feature, anti-pattern, misattributed cause.
- [ ] Interview pattern : Before implementing in unfamiliar territory, Claude asks clarifying questions about edge cases, failure modes, and non-obvious implications.
- [ ] Reasoning bias : The session that generated code retains its own reasoning and won't catch its own mistakes. Always review with a fresh independent instance.
- [ ] Tie-breaker : When two exam options seem correct, prefer the more direct, lower-effort, documented mechanism.
- [ ] Two-question filter : (1) Does this fix the root cause? (2) Is this the minimum intervention? Eliminate any exam option that fails either.

### Tools & error handling

- [ ] Empty success : Anti-pattern of returning an empty result marked as successful on failure. Masks failures; the coordinator cannot decide how to recover.
- [ ] Generic error response : Anti-pattern — "search unavailable" with no `errorCategory`, `attemptedQuery`, or `partialResults`. The coordinator has no basis to recover.
- [ ] Over-engineered classifier : Anti-pattern — building an ML routing model before trying improved tool descriptions or explicit prompt criteria.
- [ ] Silent suppression : Anti-pattern of swallowing a failure and returning an empty success. Distinguish "no results" from "search failed".
- [ ] Tool description quality : The primary mechanism by which Claude selects between similar tools. Improve descriptions before adding few-shot examples or routing classifiers.

### Information integrity

- [ ] Attribution loss : The "claim → source" link being dropped during summarization. Cannot be recovered after the fact; require structured claim-source mappings up front.
- [ ] Conflict annotation : When sources disagree, keep all values with attribution rather than silently picking one.
- [ ] Information provenance : Structured claim-source mappings — `{claim, source_url, document_name, publication_date, excerpt}` — preserved through every pipeline stage.
- [ ] Temporal mismatch : Two sources disagreeing because they describe different time periods. Include dates so it's not misread as contradiction.

### Routing & escalation

- [ ] Field-level routing : Sending a document to human review based on which specific fields are uncertain, not a single document-level score.
- [ ] Frustration ≠ explicit request : Negative sentiment is not a request for a human. Acknowledge and offer to resolve; escalate only on reiterated explicit request.
- [ ] Multi-concern request : A single user message with several distinct asks. Decompose explicitly, investigate in parallel, validate every concern is addressed before sending.
- [ ] Policy gap : A situation policy doesn't cover. Escalate — never extrapolate.
- [ ] Sentiment-based escalation : Anti-pattern — routing to a human because the customer sounds upset. Sentiment doesn't correlate with case complexity.
- [ ] Structured handoff : Self-contained JSON payload (customer ID, summary, actions taken, recommendation, escalation reason) the human operator can act on without reading the transcript.

### Quality, evaluation & review

- [ ] Aggregate-accuracy masking : A 97% headline accuracy hiding a 60% segment failure. Always stratify by document type and field before trusting an overall number.
- [ ] Attention dilution : Quality degradation when a model tracks too many distinct transformation goals or too much input at once. Multi-pass review and prompt chaining mitigate this.
- [ ] Confidence calibration : Validating model-reported or pipeline-computed confidence against a labeled validation set, segment by segment — not by intuition.
- [ ] Consensus voting : Anti-pattern of running N reviews and flagging only findings appearing in ≥M of them. Suppresses intermittently detected real bugs.
- [ ] Coverage gap : A sub-topic or section the pipeline failed to research or extract. Flag explicitly in the final output rather than burying it.
- [ ] Field-level confidence : Per-field probability scores on an extraction, used to route documents (or specific fields) to human review.
- [ ] Multi-pass review : Split a large review into per-file local passes plus a separate cross-file integration pass — avoids attention dilution.
- [ ] Self-correction pattern : Extract both stated and computed values from a document so downstream code can detect mismatches automatically.
- [ ] Stratified sampling : Drawing a labeled sample that covers each document type and field segment, including high-confidence extractions where novel error patterns hide.
- [ ] Validation-retry loop : Re-prompt the model with the original input plus specific validation errors. Effective for structural errors; ineffective when source information is simply absent.

### Security

- [ ] Defense in depth : Layered protection: delimit untrusted content, sanitize tool results in hooks, restrict write tools from content-processing agents.
- [ ] Prompt injection : Embedded instructions inside fetched content trying to hijack the agent. Mitigate with delimiters, sanitization hooks, and restricted write tools.

## Techniques & Workflows

### Context & cache

- [ ] Cache-then-shrink-then-measure : Production context-management order — (1) cache reusable prefixes, (2) compact / context-edit the rest, (3) count tokens before sending.
- [ ] Fresh session + injected summary : When the codebase has drifted (40+ files changed) since the last session, start fresh and inject a tight summary instead of resuming on stale state.
- [ ] Pinning invariants : Keep amounts, IDs, and dates outside the compactable region (e.g., in a `<case_facts>` block in the system prompt) so summarization can't approximate them.
- [ ] Pre-flight token counting : `client.messages.count_tokens(...)` before sending to confirm the request fits — counts inputs only; output cost is unknown until generated.
- [ ] Scratchpad files : Agents write key findings to a local file and re-read it before each new question — survives compaction and `/clear`.

### Session & planning

- [ ] Course-correction : Press `Esc` to stop, type the correction, continue. After two failed corrections, `/clear` and re-prompt rather than compounding confusion.
- [ ] Forking a session : Branch a session into two independent copies from a shared base. For genuine A/B comparisons; close losers promptly so context doesn't keep growing in both branches.
- [ ] Interview pattern (as workflow) : Have Claude ask clarifying questions before writing any code. Useful in unfamiliar domains or when multiple valid approaches exist.
- [ ] Plan-then-execute (workflow) : `claude --plan "investigate"`, then `claude "implement the plan"`. Plan mode for design; direct execution for implementation.

### Subagents & state

- [ ] Independent-instance review : Run the review pass in a new session with no prior context. The generator's session defends its own choices.
- [ ] Manifest export : Each subagent writes its progress to a known file path; a top-level `manifest.json` indexes status (`completed` / `in_progress` / `not_started`).
- [ ] Structured handoff payload : JSON with customer ID, issue summary, actions taken, recommendation, escalation reason — the operator must understand without the transcript.
- [ ] Structured state persistence : Write decisions and findings (not raw tool output) to per-subagent state files at meaningful checkpoints. Crash recovery loads files, not chat logs.

### Tool selection & invocation

- [ ] Fine-grained tool streaming : Streams `tool_use` parameters incrementally so the UI can render "writing…" while a long tool input is constructed.
- [ ] Forced tool selection : `tool_choice: {"type": "tool", "name": "..."}` — guarantees a specific tool runs first (e.g., extract metadata before enrichment).
- [ ] Programmatic tool calling : Multi-tool workflow inside a code-execution sandbox in one API turn. Cuts latency and tokens; cap iterations server-side or runaway cost hides.
- [ ] Splitting vs consolidating tools : Prefer 4–5 narrow tools with rich descriptions over one wide `manage_X(action, ...)`. Description quality drives selection more than capability.
- [ ] Tool description rewriting : Add purpose, accepted input formats, example queries, edge cases, and a "DO NOT use for X" line. Fix descriptions before reaching for few-shot or a classifier.
- [ ] Tool search : When a workspace has >30 tools, only names and descriptions are sent upfront; full schemas load on demand. Tradeoff: one extra discovery round-trip.

### Prompting & schema design

- [ ] Few-shot prompting : 2–4 contrastive examples for ambiguous scenarios, format consistency, or false-positive reduction. Include reasoning, not just labels.
- [ ] Nullable + present pattern : Schema design making "missing" a first-class state: `{present: false}` or nullable types — prevents hallucinated values.
- [ ] `"other"` + detail string pattern : Enum with an `"other"` value plus an `other_detail` string. Avoids collapsing every unrecognized type to "other" with no signal.

### Validation & review

- [ ] De-duplication by fingerprint : Identify review findings by `(file, line, rule_id, snippet_hash)` instead of line number alone. Survives code re-ordering.
- [ ] Error-driven loop : Collect all failures, fix in batch, re-run. Pair with a max-iteration cap so unfixable bugs don't loop forever.
- [ ] Multi-pass review (workflow) : Pass 1: per-file local issues in parallel. Pass 2: cross-file integration. Pass 3 (large PRs): cross-service contract check. De-dup against prior review.
- [ ] Re-prompting for missed concerns : After generating a response to a multi-concern request, validate every identified concern was addressed; re-prompt on misses before sending.
- [ ] Segmented routing : Specialize the pipeline for a hard segment (handwritten forms, multi-column layouts) instead of one global model handling everything.
- [ ] Stratified random sampling : Draw a labeled sample covering each segment, including high-confidence ones where novel errors hide. Alert if labeled-sample error rate exceeds threshold.
- [ ] Test-driven iteration : Write the test suite first; iterate by sharing failures with Claude. The test suite, not you, is the feedback loop.

### Hook patterns

- [ ] Async fire-and-forget hook : Hook returning `{async: true, asyncTimeout: N}` for logging or webhooks. Cannot block, modify, or inject context.
- [ ] `PostToolUse` normalization : Use a hook to rewrite tool output (Unix timestamp → ISO 8601; numeric status → label) so the model sees consistent shapes across providers.
- [ ] Trimming tool output (in a hook) : A `PostToolUse` hook keeps the 5 relevant fields out of a 40-field tool response so every subsequent turn sees the lean version.
