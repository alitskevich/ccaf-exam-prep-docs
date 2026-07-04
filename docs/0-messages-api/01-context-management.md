# Context Management API Features

Combine **automatic prompt caching**, **compaction**, and **token counting** as the production stack — each handles a distinct failure mode (cost, window overflow, oversized inputs). Apply in order: (1) **Cache** what you'll re-read, (2) **Shrink** what you'll keep (compaction, context editing), (3) **Measure** before you send (token counting).

### Key Concepts

| If you need... | Reach for | Why |
|---|---|---|
| Hands-off cache management | **Automatic prompt caching** | One parameter, no breakpoints |
| Tight control over cache layout | **Manual prompt caching** | Explicit `cache_control` blocks |
| Cache lives across user sessions | **1-hour TTL** | Longer reuse window |
| Long agent loops without context blowup | **Compaction** | Server-side summarization |
| Cheap drop of stale tool noise | **Context editing** | Surgical removal |
| "Will this fit?" guard | **Token counting** | Pre-flight check |

- Cache breakpoints invalidate when *anything before them* changes — keep volatile content (case facts, current timestamp) **after** cached blocks
- Compaction can drop a tool result you still need — pin invariants (IDs, amounts) outside the compactable region
- Automatic prompt caching is **always preferred** over manual breakpoints unless you have a measured reason to override
- **5-min vs 1-hour TTL cost model**: first hit pays full cost + a small write surcharge; subsequent hits within the TTL window are ~10% of input cost. 1-hour tier has higher write surcharge — only worth it for genuine cross-session reuse
- **Context editing** is distinct from compaction — it auto-clears stale tool results and old thinking blocks before they count against the budget. Use when 90% of past tool results are no longer needed; use compaction when *recent* content matters but you need a smaller version of it
- **Token counting endpoint** counts *input* only — output cost is unknown until generated. Use it to check headroom, not to estimate total request cost
- **All caching/compaction/context-editing/token-counting features are ZDR-eligible** ✅
- **PII / PHI redaction belongs at the orchestration layer, not in the prompt.** A pre-API pass replaces identifiers (names, MRNs, SSNs, account numbers) with stable placeholders (`PATIENT_001`, `MRN_001`); the model operates on the redacted text; a post-API pass maps tokens back when displaying to the user. The model never sees raw PHI, so logging, accidental echo, and prompt-injection leakage all lose their data source. System-prompt instructions like "do not log PHI" do not control what data reaches the model and cannot satisfy HIPAA / GDPR — the data is already in the input by the time the prompt is read

### How to

#### Token count

```ts
// Request — Anthropic.Messages.MessageCountTokensParams
// Never invokes the model — useful for budgeting before realtime or batch submission.
const count = await client.messages.countTokens({
  model: "claude-opus-4-7",
  system: [{
    type: "text",
    text: "You are a careful analyst.",
    cache_control: { type: "ephemeral", ttl: "1h" },     // mirror caching to estimate cache_creation cost
  }],
  messages: [{ role: "user", content: "Summarize Q3 invoices." }],
  tools: [/* same shape as messages.create tools — affects token count */],
  tool_choice: "auto",
  thinking: { type: "adaptive", budget_tokens: 12000 },  // affects token count
});
```

```ts
// Response — Anthropic.Messages.MessageTokensCount
{
  input_tokens: 1320,
  cache_creation_input_tokens: 240,     // would be written to cache on a real create with this cache_control
  cache_read_input_tokens: 0,           // populated when cache was previously created
}
```

- Order content so **stable prefixes are cached** and volatile parts come after
- Use **context editing** for surgical removal of stale tool results that no longer matter
- Run input through a redaction pass before `messages.create` for any field whose plaintext form is restricted (names, MRNs, SSNs, payment cards); keep the `placeholder → real` mapping server-side and reverse it on response

```ts
// Pre-API: tokenize PHI
const { redacted, mapping } = redact(payload, {
  patient_name: "PATIENT_",
  mrn: "MRN_",
  ssn: "SSN_",
});

const resp = await client.messages.create({
  model: "claude-sonnet-4-6",
  messages: [{ role: "user", content: redacted }],
});

// Post-API: restore for the user, never log the un-redacted text
const userFacing = restore(resp.content, mapping);
```

### Anti-patterns

- Manual cache breakpoints when **automatic prompt caching** would advance them for you ❌
- Letting compaction summarize transactional data — IDs and amounts get reduced to "around X" ❌
- Increasing context window to fix attention degradation — delays the problem, doesn't solve it ❌
- Skipping token counting and relying on the API to error — wastes a round trip and costs credits on rejection ❌
- 1-hour TTL by default — pay for it only when reuse is genuinely cross-session ❌
- "Tell the model not to log PHI" — a prompt instruction cannot control what data reaches the model; tokenize at the orchestration layer instead ❌
- TLS-only as a privacy story — encryption in transit protects the wire, not the payload from the model that decrypts and reads it ❌
