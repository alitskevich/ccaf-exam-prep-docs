# Batch API

Use the **Batches API** only for latency-tolerant work (overnight reports, weekend backfills) where the SLA of "up to 24 hours" is acceptable.

### Key Concepts

| Decision | Choose |
|---|---|
| PR security scan that blocks the merge button | **Real-time** — blocks UI |
| Daily dependency vulnerability report sent to security team | **Batch** — overnight is fine |
| Sprint-end code quality summary every two weeks | **Batch** — bi-weekly job |
| Live customer support response generation | **Real-time** — user is waiting |

- Batch SLA is *up to* 24h — don't promise downstream consumers anything shorter
- Real-time for "could-be-batched" workloads pays 2× for latency you don't need
- A batch with one bad request doesn't fail the whole batch — check per-request `result.type`
- Batch + timeout fallback to real-time **adds complexity without resolving the mismatch** — the original workflow's latency requirement never changed

### How to

```python
# Batch — async, 50% off, up to 24h
batch = client.messages.batches.create(requests=[
    {"custom_id": f"dep-{i}", "params": {
        "model": "claude-opus-4-7",
        "messages": [{"role": "user", "content": payload}],
        "max_tokens": 1024,
    }} for i, payload in enumerate(deps)
])

# Real-time — blocking, full price
resp = client.messages.create(
    model="claude-opus-4-7",
    messages=[{"role": "user", "content": pr_diff}],
    max_tokens=2048)
```

```typescript
// Re-submit ONLY failures — batch results carry custom_id back
const failed = (await client.messages.batches.results(batch.id))
  .filter((r) => r.error)
  .map((r) => r.custom_id);

await client.messages.batches.create({
  requests: docs.filter((d) => failed.includes(d.id)).map((d) => ({ /* ... */ })),
});

// Note: Batch API does NOT support multi-turn tool calling within one request
```

- Classify by who is waiting: human at a UI / CI gate / downstream pipeline → real-time; cron-triggered, results consumed later → batch
- For mixed workloads, split them — blocking pieces real-time, latency-tolerant pieces in batch
- Pair `custom_id` with every batch request — result order isn't guaranteed; correlation is on you
- Re-submit only failed `custom_id`s on partial failure
- Plan SLA backwards from the 24h ceiling — if consumers need 30h end-to-end, submit batches every 4h so the worst-case wait fits

#### Batch create

```ts
// Request — Anthropic.Messages.Batches.BatchCreateParams
// 50% discount vs realtime, 24h SLA. Results arrive out-of-order — correlate by custom_id, not array index.
const batch = await client.messages.batches.create({
  requests: [
    {
      custom_id: "invoice-INV-42",            // unique per-item correlation key
      params: {
        model: "claude-opus-4-7",
        max_tokens: 1024,
        messages: [{ role: "user", content: "Classify invoice INV-42." }],
      },
    },
    {
      custom_id: "invoice-INV-43",
      params: { /* same shape as messages.create params */ },
    },
  ],
});
```

```ts
// Response — Anthropic.Messages.Batches.MessageBatch
{
  id: "msgbatch_01XYZ",
  type: "message_batch",
  processing_status: "in_progress",          // "in_progress" | "canceling" | "ended"
  request_counts: { processing: 2, succeeded: 0, errored: 0, canceled: 0, expired: 0 },
  created_at: "2026-05-13T10:00:00Z",
  expires_at: "2026-05-14T10:00:00Z",
  ended_at: null,
  archived_at: null,
  cancel_initiated_at: null,
  results_url: null,                          // JSONL endpoint, populated once processing_status === "ended"
}

// One JSONL line at results_url — Anthropic.Messages.Batches.MessageBatchIndividualResponse
{
  custom_id: "invoice-INV-42",
  result: {
    type: "succeeded",                        // "succeeded" | "errored" | "canceled" | "expired"
    message: { /* Anthropic.Messages.Message — same shape as messages.create response */ },
    // error?: { type, message }  when type === "errored"
  },
}
```

### Anti-patterns

- Using batch API for blocking pre-merge checks — developers cannot wait up to 24h ❌
- Batch + timeout fallback to real-time — adds complexity without fixing the fundamental mismatch ❌
- Switching all workflows to batch for cost savings without checking latency requirements ❌
- Real-time for genuinely latency-tolerant bulk work — pays 2× for latency you don't need ❌
- Batching a blocking workflow for cost savings — a 2× real-time bill is far cheaper than a 24h merge gate ❌

### Problem: Batch API used for blocking pre-merge check

**Scope:** `claude-api`

**Problem statement:** To capture 50% cost savings, both a blocking pre-merge security check and an overnight technical debt report are switched to the Message Batches API. The pre-merge check now takes up to 24 hours, blocking merges and halting developer workflow.

**Root cause:** Message Batches API has no latency SLA — processing can take up to 24 hours. This is incompatible with any workflow where a human or automated process waits for the result before continuing.

**Solution:** Use Message Batches API only for the overnight technical debt report (latency-tolerant). Keep real-time synchronous API for the pre-merge check (blocking, human is waiting).

### Use case: Batch vs Real-Time

Assign each workflow to real-time API or Batch API. Justify.

1. PR security scan — must complete before merge button unlocks
2. Daily dependency vulnerability report — sent to security team each morning
3. Sprint-end code quality summary — generated every 2 weeks
4. Live customer support response generation

**Answers:** 1 → real-time (blocks merge UI) · 2 → batch (overnight is fine) · 3 → batch (bi-weekly job) · 4 → real-time (user is waiting).
