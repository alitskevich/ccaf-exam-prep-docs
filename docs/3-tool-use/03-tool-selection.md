# Tool Descriptions as Selection Mechanism

Tool descriptions are the primary mechanism by which models select tools. Minimal or overlapping descriptions cause unreliable routing among similar tools (e.g., `analyze_content` vs `analyze_document`). Keyword-sensitive system prompt wording can also create unintended tool associations that override well-written descriptions.

### How to

- Write descriptions that state purpose, accepted input formats, example queries, edge cases, and an explicit "use this NOT that" contrast with similar tools.
- Rename tools to eliminate functional overlap (e.g., rename `analyze_content` to `extract_web_results` so it obviously pairs with web search and can't be confused with document analysis).
- Split generic tools into purpose-specific tools with defined input/output contracts.
- Review system prompts for keyword-sensitive instructions that might override tool descriptions.
- Fix descriptions **before** adding few-shot examples — descriptions are the root cause of misrouting.
- Add a `pattern` in the JSON schema as a free, programmatic guard against mis-routing.
- Include a "DO NOT" line for any pair that overlap — it teaches routing better than any positive description.

### Anti-patterns

- Minimal descriptions like "Retrieves customer information" — model has no basis to differentiate ❌
- Adding few-shot examples before fixing descriptions — adds token overhead without addressing the root cause ❌
- Adding a routing classifier to compensate for poor descriptions — bypasses natural language understanding and adds maintenance burden ❌
- "Gets X information" is the universal anti-description — name the keys, the format, the boundary ❌

### Problem: Minimal tool descriptions cause wrong tool selection

**Scope:** `agent-sdk` · `api`

**Problem statement:** Agent calls `get_customer` when users ask about orders (e.g., "check my order #12345") instead of `lookup_order`. Both tools have minimal descriptions and accept similar identifier formats, causing 30%+ misrouting.

**Root cause:** Minimal descriptions give the model insufficient context to differentiate between similar tools. Without expressed intent, input formats, or contrast with similar tools, the model cannot choose reliably.

**Key decision:** Few-shot examples vs rich descriptions vs routing layer vs tool consolidation.

**Solution:** Expand each tool's description to include purpose, accepted input formats, example queries, edge cases, and explicit "use this NOT that" contrast with similar tools.

### Use case: Tool Description Rewrite

A customer support agent has:

- `get_customer`: "Gets customer information"
- `lookup_order`: "Gets order information"

Both accept similar identifiers. Logs show `get_customer` is called when order numbers are provided. Rewrite both descriptions to eliminate misrouting.

```python
{
  "name": "get_customer",
  "description": (
    "Look up a customer profile by customer ID or email. "
    "Inputs: customer_id like 'CUST-12345' or email like 'a@b.com'. "
    "Returns: name, email, phone, account status, plan. "
    "DO NOT call with order numbers (e.g. 'ORD-...') — use lookup_order."
  ),
  "input_schema": {
    "type": "object",
    "properties": {
      "customer_id": {"type": "string", "pattern": "^CUST-[0-9]+$"},
      "email":       {"type": "string", "format": "email"},
    },
    "oneOf": [{"required": ["customer_id"]}, {"required": ["email"]}],
  },
}

{
  "name": "lookup_order",
  "description": (
    "Look up an order by order ID. "
    "Inputs: order_id like 'ORD-5521'. "
    "Returns: items, total, status, ship-to, customer_id of buyer. "
    "DO NOT call with customer IDs or emails — use get_customer."
  ),
  "input_schema": {
    "type": "object",
    "properties": {"order_id": {"type": "string", "pattern": "^ORD-[0-9]+$"}},
    "required": ["order_id"],
  },
}
```
