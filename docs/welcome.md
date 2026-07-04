# Claude Certified Architect Foundations Certification

> **DISCLAIMER:** These materials are provided **as-is** for **educational purposes** and **personal use only**. They were compiled with the assistance of AI and **may contain mistakes or inaccuracies**. Use at your own risk.

The **Claude Certified Architect — Foundations** certification confirms that a specialist can make sound trade-off decisions when implementing real-world Claude-based solutions. The exam assesses foundational knowledge of Claude Code, the Claude Agent SDK, the Claude API, and the Model Context Protocol (MCP)—the core technologies for building production applications with Claude.

Questions are based on realistic industry scenarios: building agentic systems for customer support, designing multi-agent research pipelines, integrating Claude Code into CI/CD, creating developer productivity tools, and extracting structured data from unstructured documents.

## Overview

- Format: 60 questions, 120 minutes
- Scenarios: 4 of 6 presented at random
- All questions are **multiple choice**, each with **one correct response** and **three distractors** (response options a candidate with incomplete knowledge might choose)
- Select the single response that best completes the statement or answers the question
- **Unanswered questions are scored as incorrect.** **Guessing is zero-penalty**: if unsure, eliminate distractors and guess. Never leave a question blank.
- **Pass or fail** designation scored against a minimum standard established by subject matter experts
- Results reported as a **scaled score of 100–1,000**, emailed within 2 business days. Scaled scoring equates scores across exam forms with slightly different difficulty.
- **Minimum passing score: 720**
- Retakes: None — one attempt only

### Target Candidate

Candidates must demonstrate both conceptual knowledge and **practical judgment** about architecture, configuration, and tradeoffs in production deployments.

The ideal candidate is a **solution architect** who designs and implements production applications with Claude, with hands-on experience in:

- Building agentic applications using the Claude Agent SDK, including multi-agent orchestration, subagent delegation, tool integration, and lifecycle hooks
- Configuring and customizing Claude Code for team workflows using `CLAUDE.md` files, Agent Skills, MCP server integrations, and plan mode
- Designing Model Context Protocol (MCP) tool and resource interfaces for backend system integration
- Engineering prompts that produce reliable structured output, leveraging JSON schemas, few-shot examples, and extraction patterns
- Managing context windows effectively across long documents, multi-turn conversations, and multi-agent handoffs
- Integrating Claude into CI/CD pipelines for automated code review, test generation, and pull request feedback
- Making sound escalation and reliability decisions, including error handling, human-in-the-loop workflows, and self-evaluation patterns

> **Experience Level:** typically **6+ months** of practical experience building with Claude APIs, Agent SDK, Claude Code, and MCP, understanding both the capabilities and limitations of large language models in production environments.

### Domains

This exam tests foundational knowledge across 5 weighted domains:

1. Agentic Architecture & Orchestration — **27%**
2. Tool Design & MCP Integration — **18%**
3. Claude Code Configuration & Workflows — **20%**
4. Prompt Engineering & Structured Output — **20%**
5. Context Management & Reliability — **15%**

## Technologies and Concepts

### Claude Agent SDK

- **Agent definitions and loops:** `AgentDefinition`, control flow based on `stop_reason`, tool result handling, loop termination conditions
- **Multi-agent orchestration:** Coordinator-subagent patterns, task decomposition, parallel subagent execution, iterative refinement loops
- **Subagent context management:** Explicit context passing, structured state persistence, crash recovery using manifests
- **Subagent spawning:** `Task` tool, `allowedTools` configuration
- **Hooks:** `PostToolUse`, tool call interception
- **Tool interface design:** Effective tool descriptions, splitting vs consolidating tools, naming to reduce ambiguity
- **Error handling and propagation:** Structured error responses, transient vs business vs permission errors, local recovery before escalation

### Model Context Protocol (MCP)

- **Servers, tools, resources:** Resources for content catalogs, tools for actions, description quality for adoption
- **Tool call results:** `isError` flag, tool descriptions, tool distribution
- **Server configuration:** `.mcp.json`, project vs user scope, environment variable expansion, multi-server simultaneous access

### Claude Code

- **CLAUDE.md configuration:** Hierarchy (user/project/directory), `@import` patterns
- **Path-scoped rules:** `.claude/rules/` with YAML frontmatter and glob patterns
- **Custom commands:** `.claude/commands/` for slash commands (project vs user scope)
- **Skills:** `.claude/skills/` with `SKILL.md` frontmatter (`context: fork`, `allowed-tools`, `argument-hint`)
- **Session control:** Plan mode vs direct execution, `/memory`, `/compact`, `--resume`, `fork_session`, `Explore` subagent

#### Claude Code CLI

- `-p` / `--print` for non-interactive mode
- `--output-format json`, `--json-schema` for structured CI output

### Claude API

- `tool_use` with JSON schemas
- `tool_choice` options (`"auto"`, `"any"`, forced tool selection)
- `stop_reason` values (`"tool_use"`, `"end_turn"`)
- `max_tokens`, system prompts

### Message Batches API

- Appropriateness and latency tolerance assessment
- 50% cost savings, up to 24-hour processing window
- `custom_id` for request/response correlation
- Polling for completion
- No multi-turn tool calling support

### Structured output and validation

- **Structured output via tool_use:** Schema design, `tool_choice` configuration, nullable fields to prevent hallucination
- **JSON Schema:** Required vs optional fields, enum types, nullable fields, `"other"` + detail string patterns, strict mode for syntax error elimination
- **Pydantic:** Schema validation, semantic validation errors, validation-retry loops

### Prompting Techniques

- **Built-in tools** (Read, Write, Edit, Bash, Grep, Glob): purpose and selection criteria
- **Plan mode vs direct execution:** Complexity assessment, architectural decisions, single-file changes
- **Prompt chaining:** Sequential task decomposition into focused passes
- **Few-shot prompting:** Ambiguous scenario targeting, format consistency, false positive reduction, generalization to new patterns
- **Iterative refinement:** Input/output examples, test-driven iteration, interview pattern, sequential vs parallel issue resolution
- **Context window management:** Token budgets, progressive summarization, "lost in the middle", trimming verbose tool outputs, structured fact extraction, position-aware input ordering, scratchpad files
- **Session management:** Resume, `fork_session`, named sessions, context isolation
- **Confidence calibration:** Field-level scoring, calibration on labeled validation sets, stratified sampling for error rate measurement
- **Human review workflows:** Accuracy segmentation by document type and field
- **Escalation decision-making:** Explicit criteria, honoring customer preferences, policy gap identification
- **Information provenance:** Claim-source mappings, temporal data handling, conflict annotation, coverage gap reporting

### Out-of-Scope Topics

The following will **NOT** appear on the exam:

- Fine-tuning Claude models or training custom models
- Claude API authentication, billing, or account management
- Detailed implementation of specific programming languages or frameworks (beyond what's needed for tool and schema configuration)
- Deploying or hosting MCP servers (infrastructure, networking, container orchestration)
- Claude's internal architecture, training process, or model weights
- Constitutional AI, RLHF, or safety training methodologies
- Embedding models or vector database implementation details
- Computer use (browser automation, desktop interaction)
- Vision/image analysis capabilities
- Streaming API implementation or server-sent events
- Rate limiting, quotas, or API pricing calculations
- OAuth, API key rotation, or authentication protocol details
- Specific cloud provider configurations (AWS, GCP, Azure)
- Performance benchmarking or model comparison metrics
- Prompt caching implementation details (beyond knowing it exists)
- Token counting algorithms or tokenization specifics

## Reasoning Principles

### Answering strategy for multiple-choice questions

- Identify the scenario and the domain the question is testing
- Eliminate options that match known anti-patterns — these are always wrong
- Of remaining options, choose the one that uses the **most direct, lowest-effort, official mechanism**
- Two-question filter:
  - **Does this fix the root cause?** — If it treats a symptom (e.g., confidence threshold on a description problem), eliminate it.
  - **Is this the minimum intervention?** — If a simpler fix exists that hasn't been tried, the complex option is wrong.

### Tie-breaker when two options both seem correct

The exam rewards the option that is more direct and lower-effort, using the specific documented mechanism rather than a workaround.

- Programmatic > prompt-based (for enforcement)
- Structured context > generic signals (for errors, escalation, review criteria)
- Descriptions > routing layer (for tool selection)
- Description improvement before few-shot examples
- Few-shot > detailed instructions (for format consistency)
- Independent instance > self-review (for code review quality)
- Sync API > batch API (when blocking); Batch API > sync API (overnight / cost)

### Distractor anatomy

Every distractor is one of:

- **A real technique in the wrong context** (few-shot for tool ordering)
- **An over-engineered solution** (classifier before prompt optimization)
- **A non-existent feature** (`CLAUDE_HEADLESS`, `--batch`)
- **An anti-pattern** (empty success, generic error, consensus voting)
- **A misattributed cause** (blaming synthesis agent for coordinator decomp failure)

Eliminate any option containing these phrases without reading further:

| Distractor phrase | Why it's wrong |
|---|---|
| "enhance the system prompt to state that X is mandatory" | Prompt instructions are probabilistic — cannot enforce ordering |
| "add few-shot examples showing the agent always doing X first" | Probabilistic; insufficient for financial / safety consequences |
| "implement a routing classifier / deploy a separate classifier" | Over-engineered; requires labeled data + ML infra before prompt is tried |
| "sentiment analysis to detect frustration" | Sentiment ≠ case complexity |
| "self-report a confidence score (1–10)" | LLM self-reported confidence is poorly calibrated for escalation |
| "switch to a higher-tier model with a larger context window" | Larger windows do not fix attention quality or dilution |
| "run three independent passes and flag issues appearing in 2 of 3" | Consensus voting suppresses real bugs caught intermittently |
| "have the web search agent proactively cache" | Speculative caching cannot reliably predict what will be needed |
| "return empty result set marked as successful" | Masks failures; prevents coordinator recovery |
| "generic 'search unavailable' status after retries" | Hides context the coordinator needs for intelligent recovery |
| "propagate exception to top-level handler that terminates workflow" | Single subagent failure should not kill the whole workflow |
| "batch both workflows" when one is blocking | Batch API has no latency SLA — unacceptable for blocking pre-merge |
| `CLAUDE_HEADLESS=true` / `--batch` flag | Non-existent features |
| "redirect stdin from /dev/null" | Unix workaround, not the correct Claude Code approach |
| "place in `~/.claude/commands/`" when team-wide availability needed | User-level — invisible to teammates |
| "place in `CLAUDE.md` body" when defining a command | CLAUDE.md is for instructions, not command definitions |

Wrong-answer traps across all domains:

1. Parsing natural language to determine loop termination — use `stop_reason`
2. Arbitrary iteration caps as the primary stopping mechanism
3. Checking assistant text content as a completion indicator
4. Prompt-based enforcement for critical business rules — use hooks
5. Same-session self-review (reasoning bias is retained)
6. 18 tools per agent instead of 4–5 (degrades selection accuracy)
7. Sentiment-based escalation (sentiment ≠ complexity)
8. Aggregate accuracy metrics masking per-document-type failures
9. Generic error responses ("search unavailable") — always include structured context
10. `CLAUDE_HEADLESS` env var or `--batch` flag in CI — neither exists; use `-p`
11. Subagents inheriting parent context automatically — they don't, pass explicitly
12. Multiple Task calls across separate turns for parallelism — emit in a single response

## Preparation Recommendations

1. **Build an agent with the Claude Agent SDK** — implement a full agent loop with tool calling, error handling, and session management. Practice subagents and explicit context passing.
2. **Configure Claude Code for a real project** — use CLAUDE.md hierarchy, path-specific rules in `.claude/rules/`, skills with `context: fork` and `allowed-tools`, and MCP server integration.
3. **Design and test MCP tools** — write descriptions that differentiate similar tools, return structured errors with categories and retry flags, and test against ambiguous user requests.
4. **Build a data extraction pipeline** — use `tool_use` with JSON schemas, validation/retry loops, optional/nullable fields, and batch processing via the Message Batches API.
5. **Practice prompt engineering** — add few-shot examples for ambiguous scenarios, explicit review criteria, and multi-pass architectures for large code reviews.
6. **Study context management patterns** — extract facts from verbose outputs, use scratchpad files, and delegate discovery to subagents to handle context limits.
7. **Understand escalation and human-in-the-loop** — when to escalate (policy gaps, explicit user request, inability to make progress) and confidence-based routing workflows.
8. **Take a practice exam** before the real one. It uses the same scenarios and format.
