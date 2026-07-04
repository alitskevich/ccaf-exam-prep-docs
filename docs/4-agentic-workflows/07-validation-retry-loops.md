# Validation-Retry Loops

Append specific error feedback on retry — but retries only fix structural errors, not absent information.

### Key Concepts

- **Retry is effective for:** format mismatches, wrong field placement, structural output errors
- **Retry is NOT effective for:** information simply absent from the source document
- Tool use enforces schema syntax, so semantic errors (values don't sum, wrong field placement) persist after syntax errors are eliminated
- Add a `detected_pattern` field to each finding to track which source patterns trigger false-positive dismissals
- Give Claude something to verify against. Tests, screenshots, expected outputs. Without verification, you become the only feedback loop — and you're not as fast or thorough as a test suite
- "Field absent" must be expressible in the schema — otherwise the model invents

### How to

- On retry, send: original document + failed extraction + **specific validation errors** (not generic "try again")
- Extract `calculated_total` alongside `stated_total` to flag discrepancies automatically
- Add `conflict_detected: boolean` for internally inconsistent source data
- Cap retries by error type, not iteration count — route format errors to retry, absent-info to human review or null+reason
- If the project has tests, run them. If it doesn't, write the failing test *first*, then fix

```typescript
// Self-correction: extract redundant signals, detect inconsistency, retry with feedback
const schema = {
  line_items:       { type: "array",  items: { type: "object" } },
  calculated_total: { type: "number" },
  stated_total:     { type: "number" },
};

const extracted = await extract(doc, schema);

if (Math.abs(extracted.calculated_total - extracted.stated_total) > 0.01) {
  await extract(doc, schema, {
    feedback: `Previous extraction:
${JSON.stringify(extracted, null, 2)}

calculated_total ($${extracted.calculated_total}) does not match
stated_total ($${extracted.stated_total}). Re-examine line items —
likely a missing or duplicate row.`,
  });
}

// Don't waste retries when the info is simply absent
if (extracted.po_number === null && !doc.has("PO_section")) {
  return { needsExternalDoc: true, ask: "Provide the purchase order separately" };
}

// Track WHY a finding fired — for false-positive analysis
const finding = { ...issue, detected_pattern: "stale_param_reference" };
```

```
1. Extract → validate against schema
2. If validation fails:
   - Append: original doc + failed extraction + specific errors
   - Retry: "The previous extraction had these errors: [list]. Please correct them."
3. If info is absent from source → mark as null, route to human review
```

#### When retry will help vs when it won't

| Effective | Not effective |
|---|---|
| Format errors (date in wrong format) | Information absent from the source document |
| Structural errors (field placed in the wrong location) | Required context is in another document not provided |
| Arithmetic inconsistencies (model can re-check sums) | The source document itself is wrong |

### Anti-patterns

- Retrying when required information is absent from the source — wastes tokens with no possible improvement
- Generic retry prompt without specific error feedback — model cannot correct what it cannot see
- Treating schema-valid output as semantically correct — tool use eliminates syntax errors, not semantic errors
- If the *source document* is wrong, retry produces the same wrong number; route to human

### Use case: Retry Decision

For each failure, decide: retry with feedback, or route to human review?

1. Extracted `shipping_cost: 12.50` placed in `total_cost` field
2. Invoice total stated as $150 but line items sum to $143
3. `date_of_birth` field is empty — field does not appear anywhere in the document
4. Phone number extracted as `"555-1234"` but schema requires `"+1-555-1234"` format

```python
def route(finding):
    match finding:
        case FieldMislabeled(actual_field="shipping_cost", placed_in="total_cost"):
            return retry(feedback="`shipping_cost` ≠ `total_cost`. Re-extract.")     # 1

        case TotalMismatch(stated=t, sum_of_lines=s) if abs(t-s) > 0.01:
            return human_review("Possible source-document error: "
                                f"stated total {t} ≠ line sum {s}")                  # 2

        case FieldAbsent(field="date_of_birth", in_source=False):
            return write_null(field="date_of_birth", reason="not present")           # 3

        case FormatMismatch(value="555-1234", expected="+1-555-1234"):
            return retry(feedback="Add country code prefix: '+1-555-1234'.")         # 4
```

**Suggested answers:** 1 → retry · 2 → human · 3 → null + reason · 4 → retry.
