# Checkpointing and Cost Tracking

File checkpointing lets you rewind file changes mid-session by user-message UUID. Cost tracking comes from `ResultMessage.total_cost_usd` (per-call estimate) plus per-step `usage` fields — but the **Usage and Cost API** is the authoritative billing source.

### Key Concepts

| Aspect | Detail |
|---|---|
| Tracked operations | `Write`, `Edit`, `NotebookEdit` |
| **NOT** tracked | Bash-driven changes, directory create/delete |
| Scope | Same session only; local files only |
| Effect of rewind | Restores **files**, not conversation history |

- File checkpointing requires `enableFileCheckpointing: true` AND `extraArgs: { "replay-user-messages": null }` to receive UUIDs
- Capture the user-message `uuid` from the stream — that's the checkpoint ID for `rewindFiles(uuid)`
- **Parallel tool calls** produce multiple assistant messages with the same `id` — deduplicate by ID before summing token usage

### How to

```typescript
const opts = {
  enableFileCheckpointing: true,
  permissionMode: "acceptEdits" as const,
  extraArgs: { "replay-user-messages": null }   // required to receive UUIDs
};

let checkpointId: string | undefined;
let sessionId: string | undefined;

const response = query({ prompt: "Refactor auth module", options: opts });

for await (const m of response) {
  if (m.type === "user" && m.uuid && !checkpointId) checkpointId = m.uuid;
  if ("session_id" in m) sessionId = m.session_id;
}

// Later — resume + rewind
if (checkpointId && sessionId) {
  const rewindQuery = query({
    prompt: "",   // empty prompt opens connection
    options: { ...opts, resume: sessionId }
  });
  for await (const _ of rewindQuery) {
    await rewindQuery.rewindFiles(checkpointId);
    break;
  }
}
```

CLI equivalent:

```bash
claude -p --resume <session-id> --rewind-files <checkpoint-uuid>
```

**Checkpoint patterns:**

- **One restore point** — capture only the *first* user message UUID; rewinding restores all files to original state
- **Multiple restore points** — store every user message UUID with metadata; rewind to any
- **Pre-risk checkpoint** — overwrite `safeCheckpoint` before each turn; on detection of error, `rewindFiles` immediately and break

### Cost tracking

```typescript
// Per-call total
for await (const message of query({ prompt: "Summarize this project" })) {
  if (message.type === "result") {
    console.log(`Total cost: $${message.total_cost_usd}`);
  }
}

// Per-step token tracking — DEDUPE by message id
const seenIds = new Set<string>();
let totalInput = 0, totalOutput = 0;
for await (const message of query({ /* ... */ })) {
  if (message.type === "assistant") {
    const id = message.message.id;
    if (!seenIds.has(id)) {
      seenIds.add(id);
      totalInput  += message.message.usage.input_tokens;
      totalOutput += message.message.usage.output_tokens;
    }
  }
}

// Per-model breakdown — separates Sonnet/Opus/Haiku spending in advisor mode
if (message.type === "result") {
  for (const [model, usage] of Object.entries(message.modelUsage)) {
    console.log(`${model}: $${usage.costUSD.toFixed(4)}`);
    console.log(`  in/out/cacheRead/cacheCreate:`,
      usage.inputTokens, usage.outputTokens,
      usage.cacheReadInputTokens, usage.cacheCreationInputTokens);
  }
}
```

- The SDK **has no session-level total** — sum `total_cost_usd` yourself across calls
- Both success and error `ResultMessage`s carry `usage` and `total_cost_usd` — always read regardless of `subtype`
- For longer cache reuse across short sessions with > 5 min gaps, set `ENABLE_PROMPT_CACHING_1H: "1"` in `options.env` (higher write cost, more cache hits)

### Anti-patterns

- Relying on `total_cost_usd` for billing instead of the Usage and Cost API ❌
- Summing `usage` across assistant messages without deduplicating by `id` ❌
- Treating file checkpoints as a substitute for git ❌
- Expecting `rewindFiles` to undo conversation history — it restores files only ❌
- Forgetting `replay-user-messages: null` in `extraArgs` — without it, no UUIDs reach you ❌
