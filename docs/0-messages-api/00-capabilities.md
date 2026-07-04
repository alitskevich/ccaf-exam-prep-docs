# Messages API Capabilities Selection

Pick API capabilities by the failure mode they prevent: large inputs, variable cost, parsing errors, attribution loss, residency.

### Key Concepts

| If you need... | Reach for | Why |
|---|---|---|
| Huge inputs in one shot | **Context windows (1M)** | Avoids manual chunking |
| Variable difficulty, prod traffic | **Adaptive thinking** | Pays only when needed |
| Inspectable reasoning | **Extended thinking** | Trace + final answer |
| One knob, less complexity | **Effort** | Coarse but easy |
| Provable JSON shapes | **Structured outputs** | Replace post-hoc parsing |
| Source-grounded answers | **Citations** | Sentence-level refs |
| RAG over private data | **Search results** | Web-search-quality on your store |
| Mixed text + visual docs | **PDF support** | Native, no OCR pipeline |
| Cheap bulk, no rush | **Batch processing** | 50% off, 24h |
| Regional compliance | **Data residency** | Per-request geo pin |

- 1M context is *available*, not free — token bills scale linearly; pair with caching or compaction
- Adaptive thinking can spend its full budget on easy prompts if the system prompt nudges "always think carefully" — keep it neutral
- `inference_geo` constrains routing; some features (e.g. certain server-side tools) may not be available in every region
- **Effort knob** (`"low" / "medium" / "high" / "xhigh" / "max"`) is the simplest way to dial response thoroughness — Opus 4.7 recommends `"xhigh"`; TS SDK defaults to `"high"`, Python defers to model default
- **ZDR (Zero Data Retention)** is enabled per-organization, not a request-time flag. Per-feature ZDR support: structured outputs ✅, citations ✅, web search ✅ (except dynamic filtering), code execution ❌, Files API ❌, Skills ❌, MCP connector ❌

### How to

#### Message create — Anthropic.Messages.MessageCreateParams

```ts
const res = await client.messages.create({
  model: "claude-opus-4-7",                  // also: "claude-sonnet-4-6", "claude-haiku-4-5"
  max_tokens: 4096,                          // output budget; also a stop_reason value when hit
  effort: "xhigh",                           // "low" | "medium" | "high" | "xhigh" | "max" — Opus 4.7 recommends xhigh
  temperature: 1.0,
  stop_sequences: ["END"],
  stream: false,                             // true ⇒ SSE stream (see streaming chapter)
  metadata: { user_id: "uuid-or-hash" },     // abuse signal — no PII
  service_tier: "auto",                      // "auto" | "standard_only"
  inference_geo: "eu",                       // region pin for data residency

  output_config: { format: { type: "json_schema", schema: invoiceSchema } },

  system: [{
    type: "text",
    text: "You are a careful analyst.",
    cache_control: { type: "ephemeral", ttl: "1h" },   // prompt caching; ttl: "5m" | "1h"
  }],

  thinking: { type: "adaptive", budget_tokens: 12000 }, // adaptive reasoning spend on hard prompts

  context_management: {
    context_window: "1M",                                // huge inputs in one shot
    compaction: { trigger_tokens: 150000 },              // server-side conversation summarization
  },

  tool_choice: "auto",                       // "auto" | "any" | "none" | { type: "tool", name: "..." }

  tools: [
    {                                        // custom tool — input_schema is JSON Schema (required, enum, nullable, "other"+detail)
      name: "get_invoice",
      description: "Look up an invoice by id.",
      input_schema: {
        type: "object",
        properties: {
          id: { type: "string" },
          status: { type: "string", enum: ["open", "paid", "other"] },
        },
        required: ["id"],
      },
    },
    { type: "web_search_20250903", name: "web_search" }, // server tools: web_search, web_fetch,
                                                         // code_execution, advisor, bash, text_editor,
                                                         // memory, computer_use
  ],

  messages: [
    { role: "user", content: "Summarize Q3 invoices." },
    { role: "assistant", content: [
      { type: "tool_use", id: "toolu_01", name: "get_invoice", input: { id: "INV-42" } },
    ]},
    { role: "user", content: [{
      type: "tool_result",
      tool_use_id: "toolu_01",                  // links result back to the tool_use that requested it
      content: '{"status":"paid"}',
      is_error: false,                          // true ⇒ tool failed; model can self-correct next turn
    }]},
  ],
});
```

#### Response — Anthropic.Messages.Message

```ts
{
  id: "msg_01ABCD",
  type: "message",
  role: "assistant",
  model: "claude-opus-4-7",
  content: [
    { type: "thinking", thinking: "..." },                                  // reasoning trace
    { type: "text", text: "Summary..." },                                   // assistant prose
    { type: "tool_use", id: "toolu_02", name: "get_invoice", input: {} },   // next tool call
  ],
  stop_reason: "end_turn",       // "end_turn" | "tool_use" | "max_tokens" | "stop_sequence" | "refusal"
                                 // refusal ⇒ do NOT retry the same prompt
  stop_sequence: null,
  usage: {
    input_tokens: 1320,
    output_tokens: 480,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
    server_tool_use: { web_search_requests: 0 },
  },
}
```

### Anti-patterns

- Skipping caching on a high-traffic endpoint — a single one can 10× your monthly cost overnight ❌
- Treating beta features as production-stable — they may change without major-version notice ❌
- Compaction without pinning invariants — IDs and amounts get summarized away from chat turns ❌
- Assuming ZDR covers every feature — check the per-feature ZDR matrix before architecting ❌
