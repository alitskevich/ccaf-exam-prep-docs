# Tool Error Propagation

Structured error context enables intelligent recovery — generic failures prevent coordinators from making appropriate decisions.

### Key Concepts

| Category | `isRetryable` | Example | Agent action |
|---|---|---|---|
| **transient** | true | HTTP 503, timeout | Retry with backoff |
| **validation** | false | Invalid input format | Fix input, retry differently |
| **business** | false | Refund > policy threshold | Communicate to user, escalate |
| **permission** | false | Unauthorized access | Escalate, do not retry |

- MCP tools communicate failure via the `isError` flag; include `errorCategory`, `isRetryable`, `description`, `attemptedQuery`, and `partialResults`
- **Access failure** (HTTP 503) ≠ **valid empty result** (query succeeded, no matching records) — these must be distinguished
- Subagents attempt local recovery for transient failures first; propagate only errors they cannot resolve
- Include partial results and attempted alternatives so the coordinator can retry, escalate, or synthesize partial output
- The coordinator owns the retry cap — `isRetryable: true` without a max-retry count loops forever
- `permission_denied` messages must not leak the protected data they were trying to fetch

```json
{
  "isError": true,
  "errorCategory": "transient",
  "isRetryable": true,
  "description": "Order lookup service returned HTTP 503. Retry after 2 seconds.",
  "attemptedQuery": "order_id: ORD-9921",
  "partialResults": null
}
```

#### Structured subagent error with partial results

When a search or analysis subagent fails partway, return what it has plus alternatives:

```json
{
  "status": "partial_failure",
  "failure_type": "timeout",
  "attempted_query": "AI impact on music industry 2024",
  "partial_results": [
    {"title": "AI Music Generation Report", "url": "...", "relevance": 0.8}
  ],
  "alternative_approaches": [
    "Try a narrower query: 'AI music composition tools'",
    "Use an alternative data source"
  ],
  "coverage_impact": "Not covered: AI impact on music production"
}
```

#### Coverage annotations in final synthesis

When the coordinator proceeds despite partial coverage, flag the gap rather than burying it:

```markdown
### AI Impact on Creative Industries

#### Visual Arts (FULL COVERAGE)
[research results]

#### Music (PARTIAL COVERAGE — search agent timeout)
[partial results]
⚠️ Note: coverage for this section is limited due to a timeout in the search agent.

#### Film (FULL COVERAGE)
[research results]
```

### Anti-patterns

| Anti-pattern | Problem | Correct approach |
|---|---|---|
| Generic status "search unavailable" | Coordinator can't decide how to recover | Return error type, query, partial results, alternatives |
| Silent suppression (empty result = success) | Coordinator thinks there were no matches, but it was a failure | Distinguish "no results" from "search failure" |
| Aborting the whole workflow on one failure | You lose all partial results | Continue with partial results; annotate gaps |
| Infinite retries inside a subagent | Latency and wasted resources | Local recovery (1–2 retries), then propagate to coordinator |
| Conflating permanent errors (e.g. `not_found`) with `transient` | Endless retries on errors that will never succeed | Classify category accurately; only `transient` is retryable |

### Problem: Subagent timeout must propagate with structured context

**Scope:** `agent-sdk`

**Problem statement:** Web search subagent times out on query "AI regulation in Southeast Asia 2024." It has partial results for Thailand and Vietnam but returns a generic "search unavailable" status.

**Root cause:** No structured error contract between subagent and coordinator — generic failure messages provide no context for recovery decisions.

**Key decision:** Structured error context vs generic status vs empty success vs terminate workflow.

**Solution:** Return structured error with `failureType`, `attemptedQuery`, `partialResults`, and `potentialAlternatives` so the coordinator can retry with a modified query, proceed with partial results, or try an alternative source.

```json
{
  "status": "error",
  "failure_type": "timeout",
  "attempted_query": "AI regulation Southeast Asia 2024",
  "partial_results": { "thailand": "...", "vietnam": "..." },
  "potential_alternatives": [
    "Break query into per-country searches",
    "Try regional regulatory body databases directly"
  ]
}
```

### Use case: Error Response Design

Design structured error responses for these failure scenarios in a customer support MCP tool:

1. `get_customer` times out after 5 seconds
2. `process_refund` called with an order ID that doesn't exist
3. `process_refund` called for €750 (policy limit is €500)
4. `get_customer` called by an agent role lacking PII read permission

```json
[
  { "errorCategory": "transient",
    "isRetryable": true,
    "description": "get_customer timed out after 5s; downstream CRM unresponsive.",
    "coordinatorAction": "retry up to 2x with backoff; then escalate as 'system unavailable'." },

  { "errorCategory": "not_found",
    "isRetryable": false,
    "description": "Order ID 'ORD-9999' does not exist.",
    "coordinatorAction": "ask user to re-confirm the order number; do not retry." },

  { "errorCategory": "policy_violation",
    "isRetryable": false,
    "description": "Refund of €750 exceeds €500 cap; requires human approval.",
    "coordinatorAction": "escalate to human queue 'refunds-large'; tell user it's been routed." },

  { "errorCategory": "permission_denied",
    "isRetryable": false,
    "description": "Caller lacks 'pii:read' scope.",
    "coordinatorAction": "do not retry; do not surface PII; tell user this needs a senior agent." }
]
```
