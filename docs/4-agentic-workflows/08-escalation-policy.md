# Escalation Patterns

Honor explicit human requests immediately; escalate on policy gaps; resolve autonomously when policy is clear and progress is possible.

### Key Concepts

- **Honor immediately:** explicit human request → escalate without attempting resolution
- **Frustration is not a request:** acknowledge and offer to resolve; escalate only if reiterated
- **Policy gap:** when policy is silent, escalate — do not extrapolate
- **Multiple matches:** request additional identifiers; never select by heuristic
- **Structured handoff:** human agents may not see the transcript
- **Scope escalation:** distinct from user-driven escalation. When the agent itself discovers that the work exceeds its authorization (e.g., the user approved 2 files but a consistent refactor needs 8), it must pause, surface the discovered scope with reasons, and wait for re-authorization — never expand autonomously

### How to

- Add explicit escalation criteria with few-shot examples to the system prompt
- Distinguish frustration (acknowledge + offer to resolve) from explicit human request (escalate immediately)
- Enforce financial thresholds via a hook, not just a prompt

#### Escalation triggers

| Situation | Action |
|---|---|
| Customer explicitly asks "get me a manager" | Escalate immediately; do not attempt to solve |
| Policy does not cover the request | Escalate (e.g., competitor price matching when policy is silent) |
| Agent cannot make progress | Escalate after a reasonable number of attempts |
| Financial operation above a threshold | Escalate (preferably enforced via a hook, not a prompt) |
| Multiple matches when searching for a customer | Ask for additional identifiers; do not guess |
| Discovered work exceeds the authorized scope | Pause, report the scope expansion with file/operation list, wait for re-authorization |

#### Unreliable triggers (do not use)

| Unreliable method | Why it fails |
|---|---|
| Sentiment analysis | Customer mood does not correlate with case complexity |
| Model self-rated confidence (1–10) | Model can be confidently wrong; calibration is poor |
| An automatic classifier | Overengineering; may require training data you don't have |

#### Escalation patterns

**Immediate escalation:**

```
Customer: "I want to speak to a manager"
Agent:    [immediately calls escalate_to_human]
NOT:      "I can help with your issue, let me..."
```

**Escalation after an attempt to resolve:**

```
Customer: "My refrigerator broke two days after purchase"
Agent:    [checks the order, offers a warranty replacement]
If the customer is not satisfied → escalate
```

**Nuanced escalation (acknowledge → resolve → escalate on reiteration):**

```
Customer: "This is outrageous, I'm very unhappy with the quality!"
Agent:    [acknowledges frustration] "I understand your frustration."
          [offers resolution] "I can offer a replacement or a refund."
Customer: "No, I want to talk to someone!"
Agent:    [customer insists again → immediate escalation]
```

**Escalation for a policy gap:**

```
Customer: "Competitor X has this item 30% cheaper—give me a discount"
Policy:   covers price adjustments only on your own site
Agent:    [escalates — policy does not cover competitor price matching]
```

#### Structured handoff protocols

The summary must be self-contained — the operator does not see the full transcript:

```json
{
  "customer_id": "CUST-12345",
  "customer_name": "Ivan Petrov",
  "issue_summary": "Refund request for a damaged item",
  "order_id": "ORD-67890",
  "root_cause": "Item arrived damaged; photos attached",
  "actions_taken": [
    "Verified customer via get_customer",
    "Confirmed order via lookup_order",
    "Offered a standard replacement — customer insists on a refund"
  ],
  "refund_amount": "$89.99",
  "recommended_action": "Approve a full refund",
  "escalation_reason": "Customer requested to speak with a manager"
}
```

### Anti-patterns

- Sentiment-based escalation: negative sentiment ≠ case complexity ❌
- Self-reported confidence threshold: LLM confidence is poorly calibrated ❌
- Investigating before honoring an explicit human request ❌
- Extrapolating policy when policy is silent ❌
- Escalating without structured handoff context ❌
- Soft thresholds in prose ("be careful with refunds") get crossed; use hard numeric limits (e.g., €500) ❌
- Omitting an explicit ambiguity rule for multi-match lookups: the model picks one record and corrupts the wrong account ❌
- "Try to resolve first" without a max-attempts cap: turns into endless self-resolution; pair with a hook ❌
- Autonomous scope expansion ("the end justifies the means") — when the discovered work exceeds the user's authorization, pause and re-ask; don't push through ❌

### Problem: Wrong escalation decisions drop first-contact resolution

**Scope:** `agent-sdk`

**Problem statement:** Agent achieves 55% first-contact resolution. Logs show it escalates standard damage replacements (policy is clear) while attempting to handle policy exceptions (genuine gaps) autonomously.

**Root cause:** Unclear escalation decision boundaries — agent has no basis to distinguish "within policy" from "policy gap."

**Key decision:** Explicit criteria + few-shot vs confidence score vs sentiment analysis vs separate classifier.

**Solution:** Add explicit escalation criteria with few-shot examples directly to the system prompt. This is the proportionate fix before adding infrastructure.

**Anti-pattern:** Confidence score (poorly calibrated proxy); sentiment analysis (wrong proxy — sentiment ≠ complexity); separate classifier (requires labeled data and ML infra — over-engineered as a first step).

### Use case: Escalation Criteria

Write the system prompt escalation section for a customer support agent. Include explicit conditions for immediate escalation, conditions for offering to resolve before escalating, at least 2 few-shot examples (one each), and how to handle multiple customer identifier matches.

```markdown
## Escalation policy

### Escalate immediately (no resolution attempt)
- Refund > €500
- Account closure / GDPR data request
- Threats of legal action or harm
- Multiple customer matches for the same identifier (ambiguity → human)

### Try to resolve first; escalate if user declines or issue persists
- Address change, password reset, plan change within current entitlement
- One refund ≤ €500 with clear damage/error
- Billing question with a documented answer

### Examples

#### Immediate escalation
User: "I want to close my account and delete all my data."
You: "I'll route this to our compliance team — they handle account closure
and data deletion. Reference: ESC-{ticket}. They'll reach you within 1 business day."

#### Resolve first
User: "My package arrived damaged, order #1234, total €89."
You: "Sorry to hear that. I can issue a full refund of €89 to your original
payment method now — would you like me to proceed?"

### Multiple identifier matches
If `lookup_customer(email)` returns >1 record, DO NOT pick one.
Reply: "I see multiple accounts under that email — could you share the
account ID (CUST-...) or the last 4 digits of your payment method?"
```
