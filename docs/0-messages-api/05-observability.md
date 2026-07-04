# Observability

Wire the SDK to OpenTelemetry via env vars — three independent signals (metrics, logs, traces). For live progress UI, render the agent's `TodoWrite` payload.

### Key Concepts

| Signal | Enable with |
|---|---|
| **Metrics** — token/cost counters, sessions, tool decisions | `OTEL_METRICS_EXPORTER` |
| **Log events** — prompts, API requests/errors, tool results | `OTEL_LOGS_EXPORTER` |
| **Traces (beta)** — spans for interactions, model requests, tools, hooks | `OTEL_TRACES_EXPORTER` + `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` |

**Span hierarchy:**

```
claude_code.interaction               (one turn of the agent loop)
├── claude_code.llm_request           (each Claude API call)
└── claude_code.tool                  (each tool invocation)
    ├── claude_code.tool.blocked_on_user
    ├── claude_code.tool.execution
    └── claude_code.hook              (each hook execution; needs ENABLE_BETA_TRACING_DETAILED=1)
```

- The SDK runs Claude Code as a subprocess; OTel instrumentation lives in the CLI — configuration is **environment variables**, not options
- **TS quirk:** `options.env` *replaces* the inherited environment. Spread `...process.env` to keep `PATH`, API key, etc.
- **Python quirk:** `env` is *merged* on top of inherited environment
- **Subagent spans nest under the parent's `claude_code.tool` span** — full delegation chain in one trace
- **Never set `OTEL_*_EXPORTER=console`** in the SDK — stdout is the SDK's message channel; use a local OTel collector or Jaeger container

### How to

```typescript
const otelEnv = {
  CLAUDE_CODE_ENABLE_TELEMETRY: "1",
  CLAUDE_CODE_ENHANCED_TELEMETRY_BETA: "1",
  OTEL_TRACES_EXPORTER: "otlp",
  OTEL_METRICS_EXPORTER: "otlp",
  OTEL_LOGS_EXPORTER: "otlp",
  OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf",
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://collector.example.com:4318",
  OTEL_EXPORTER_OTLP_HEADERS: "Authorization=Bearer your-token"
};

for await (const m of query({
  prompt: "List files",
  options: { env: { ...process.env, ...otelEnv } }
})) { /* ... */ }
```

**Tag your agent for grouping in dashboards:**

```typescript
options: {
  env: {
    ...process.env,
    OTEL_SERVICE_NAME: "support-triage-agent",
    OTEL_RESOURCE_ATTRIBUTES: "service.version=1.4.0,deployment.environment=production"
  }
}
```

**Flush from short-lived calls** (default intervals: metrics 60s, traces/logs 5s):

```typescript
{ OTEL_METRIC_EXPORT_INTERVAL: "1000",
  OTEL_LOGS_EXPORT_INTERVAL:   "1000",
  OTEL_TRACES_EXPORT_INTERVAL: "1000" }
```

**Link to your application's traces** — the SDK auto-injects `TRACEPARENT` / `TRACESTATE` into the subprocess. If you call `query()` while an OTel span is active, the agent run becomes a child of your span.

### Sensitive content (opt-in only)

| Variable | Adds |
|---|---|
| `OTEL_LOG_USER_PROMPTS=1` | Prompt text |
| `OTEL_LOG_TOOL_DETAILS=1` | Tool input args (paths, commands) |
| `OTEL_LOG_TOOL_CONTENT=1` | Full tool input/output (60KB cap) |
| `OTEL_LOG_RAW_API_BODIES` | Full Messages API JSON (`1` inline 60KB; `file:<dir>` untruncated) |

Leave unset unless your pipeline is approved for the data.

### Live progress with `TodoWrite`

The agent uses the built-in `TodoWrite` tool to track progress on multi-step tasks (3+ distinct actions). Render the latest payload as your single source of truth:

```typescript
for await (const message of query({
  prompt: "Optimize my React app performance and track progress with todos",
  options: { maxTurns: 15 }
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "tool_use" && block.name === "TodoWrite") {
        const todos = block.input.todos as Array<{
          content: string;
          activeForm: string;
          status: "pending" | "in_progress" | "completed";
        }>;

        todos.forEach((t, i) => {
          const icon = t.status === "completed" ? "✅"
                    : t.status === "in_progress" ? "🔧" : "❌";
          const text = t.status === "in_progress" ? t.activeForm : t.content;
          console.log(`${i + 1}. ${icon} ${text}`);
        });
      }
    }
  }
}
```

Each todo has `content` (imperative, "Run tests"), `activeForm` (present continuous, "Running tests"; shown while `in_progress`), and `status` (`pending` / `in_progress` / `completed`).

### Anti-patterns

- `OTEL_*_EXPORTER=console` in the SDK — collides with stdout message channel ❌
- TS `options.env` without `...process.env` — strips `PATH`, API key ❌
- `OTEL_LOG_USER_PROMPTS` / `OTEL_LOG_TOOL_CONTENT` without an approved data policy ❌
- Short-lived runs without short export intervals — finishes before the 60s metrics flush ❌
