# Human Review Workflows and Confidence Calibration

Route extractions to human review based on field-level confidence scores calibrated against labeled validation sets — never trust aggregate accuracy alone.

### Key Concepts

- **97% aggregate accuracy** can hide 70% on handwritten forms and 40% on multi-column layouts — validate by document type AND field segment
- Calibrate field-level confidence against **labeled validation sets**, not intuition or model self-reported values
- Stratified random sampling must cover **high-confidence** extractions too — that's where novel error patterns hide
- Route to human review when any required field is below threshold, OR the source is ambiguous/contradictory
- Field-level routing requires **per-field confidence scores** — a single document-level score cannot drive it

### How to

- Add `field_confidence` alongside each extracted field in the schema
- Set routing thresholds per document type from labeled validation sets — not a single global threshold
- Analyze accuracy by document type and field segment before reducing human review volume
- Alert on regression: if the labeled-sample error rate in the high-confidence bucket exceeds a threshold (e.g., 2%), pause auto-approval until the cause is found

```typescript
// Aggregate metrics LIE — always stratify
const overall = 0.97;                       // looks great
const byType = {
  invoice:          0.99,
  contract:         0.98,
  handwritten_form: 0.62,                   // hidden disaster — masked by volume
};

// Stratified random sampling — catches NEW error patterns in the high-conf bucket
const sample = stratifiedSample(highConfidenceExtractions, {
  strata: { byType, byField: ["amount", "date", "party"] },
  n: 50,
});
const labeled   = await humanLabel(sample);
const errorRate = labeled.filter((s) => s.wrong).length / labeled.length;

if (errorRate > 0.02) {
  console.warn("High-conf bucket regressing — pause auto-approval");
}
```

```json
{
  "invoice_number": {"value": "INV-9921", "confidence": 0.98},
  "total_amount":   {"value": 1250.00,    "confidence": 0.72},
  "line_items":     {"value": [],         "confidence": 0.45}
}
```

```python
THRESHOLD = 0.80
if any(data[f"{field}_confidence"] < THRESHOLD for field in required_fields):
    route_to_human_review(data, document)
```

### Anti-patterns

- Trusting 97% aggregate accuracy to justify removing human review
- Reducing human review before validating accuracy per document type and field segment
- Setting a single global confidence threshold without per-segment calibration
- Sampling only low-confidence extractions — novel errors also appear in high-confidence batches
- Routing an entire segment to humans without checking the math against the review budget cap
- Spot-checking under 1% on high-volume segments — too sparse to detect drift; aim for 0.5–1% with statistical alerting

### Problem: Aggregate accuracy masks segment-level failures

**Scope:** `api`

**Problem statement:** Pipeline shows 97% aggregate accuracy. Stakeholders propose removing human review. Segment analysis reveals 68% on handwritten forms and 74% on multi-column contracts.

**Root cause:** Aggregate metric averages over high-volume, high-accuracy document types, masking poor performance on minority segments.

**Key decision:** Trust aggregate metric vs segment by document type vs add per-field confidence scoring vs reduce review incrementally per segment.

**Solution:** Segment accuracy by document type and field. Run stratified random sampling on high-confidence extractions. Only reduce review after per-segment validation passes for that segment.

### Use case: Human Review Routing

Your extraction pipeline has these stats:

| Document type | Accuracy | Volume |
|---|---|---|
| Standard invoices | 99% | 150k/month |
| Handwritten forms | 68% | 5k/month |
| Multi-column contracts | 74% | 20k/month |
| Digital receipts | 97% | 80k/month |

Management wants to cap human review at 5% of total volume. Design a routing strategy that identifies which segments to auto-process, which require full review, and how to implement field-level routing for the middle tier.

```python
TOTAL = 150_000 + 5_000 + 20_000 + 80_000          # 255k/month
HUMAN_BUDGET = 0.05 * TOTAL                         # 12,750/month

def route(doc):
    if doc.type == "handwritten":
        return "human_full"                         # 5,000 — all reviewed

    if doc.type == "multi_column_contract":
        # field-level routing within this segment
        fields = extract(doc)
        risky  = [f for f in fields if f.confidence < 0.85
                                     or f.field in CRITICAL_FIELDS]
        return "human_fields" if risky else "auto"  # ~7,750 budget remaining

    if doc.type in {"standard_invoice", "digital_receipt"}:
        # spot-check 1% as a quality signal, auto-process the rest
        return "human_spot_check" if random() < 0.01 else "auto"

CRITICAL_FIELDS = {"total", "tax", "vendor_id", "due_date"}
```

```text
Budget allocation
─────────────────────────────────────────────────────
handwritten — full review            5,000   →  39%
contracts  — field-level review     ~6,000   →  47%
spot checks (invoices+receipts)     ~2,300   →  18%
─────────────────────────────────────────────────────
total                              ~13,300   ≈ 5.2%
```
