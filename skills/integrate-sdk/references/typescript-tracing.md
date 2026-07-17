# NeoSigma TypeScript tracing integration

Instrument a TypeScript/Node codebase with the npm package `neosigma-sdk` so
agent runs land in NeoSigma as traces. This file is the API surface of record
for the TypeScript SDK (0.5.x). Parameters and exports not listed here do not
exist. If something seems missing, check https://docs.neosigma.ai/sdk/tracing
rather than guessing.

The TypeScript SDK does full agent tracing (turns, framework adapters, dual
export), not only product events. Product events (`capture`/`identify`) are in
[typescript-events.md](typescript-events.md); the two share one process and one
id model, so instrument tracing here and bind events there.

## 1. Assess the codebase

Find, in this order:

- The real execution path: the route, job, queue consumer, or CLI command that
  runs the agent. Instrument that, not a new demo.
- The LLM surface, which decides section 4: Vercel AI SDK (`ai`), LangChain
  (`@langchain/*`), the Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`),
  Anthropic Managed Agents (`@anthropic-ai/sdk` beta sessions), or raw model
  clients.
- Existing OpenTelemetry: any `NodeTracerProvider`/`BasicTracerProvider`
  construction or `trace.setGlobalTracerProvider(...)`. This decides section 5.
- The app's own ids: conversation id, end-user id, and any per-request or
  per-run id (these become `sessionId`, `distinctId`, `turnId`).
- Runtime and module system: long-running server, serverless, or CLI (decides
  the flush strategy, section 2); ESM vs CommonJS; Node or Bun.

```bash
rg -n "from \"ai\"|@ai-sdk|@langchain|claude-agent-sdk|@anthropic-ai/sdk|setGlobalTracerProvider|NodeTracerProvider|experimental_telemetry" .
```

Runtime facts: the SDK is ESM and ships OpenTelemetry JS 2.x, which requires
Node 18.19+ or 20.6+. It runs under Bun (verified on Bun 1.3). On CommonJS,
load it in an async context with `const neosigma = await import("neosigma-sdk")`
(top-level `await` is not available in CommonJS).

## 2. Install and lifecycle

```bash
npm install neosigma-sdk
```

```ts
import { init, shutdown } from "neosigma-sdk";

init(); // once at startup; reads NEOSIGMA_API_KEY; idempotent, never throws
// ... run the agent ...
await shutdown(); // once at process exit; flushes buffered spans
```

The `init` options (no other keys exist; there is NO `tracer`, `instrument`, or
`endpoint` option):

```ts
function init(overrides?: {
  apiKey?: string;
  project?: string;
  tracingEnabled?: boolean;            // auto-instrumentation flag; a NO-OP in 0.5.0 (section 4)
  attachToExistingProvider?: boolean;  // dual export, section 5
  eventsEndpoint?: string;
  otelEndpoint?: string;               // MUST be the full path ending /v1/traces
  consoleExport?: boolean;
  enabled?: boolean;
  extraSpanProcessors?: SpanProcessor[]; // dual export, section 5
  settings?: Partial<Settings>;        // any other setting by its camelCase name
}): void;
```

- `init()` is idempotent and fail-open. It never throws into app startup; on any
  failure it degrades to inactive and the app runs unchanged.
- Without an API key (and without `consoleExport`) every SDK call is a no-op, so
  the integration is safe to merge before keys are provisioned. Ids are still
  generated and the app runs normally.
- `otelEndpoint` is the FULL OTLP path. Its default is
  `https://otel.neosigma.ai/v1/traces`. If you set it from a base URL, append
  `/v1/traces` yourself, or exports 404.

Environment variables (prefix `NEOSIGMA_`, `SCREAMING_SNAKE_CASE`):
`NEOSIGMA_API_KEY`, `NEOSIGMA_PROJECT` (default `default`),
`NEOSIGMA_SERVICE_NAME` (defaults to project), `NEOSIGMA_OTEL_ENDPOINT`,
`NEOSIGMA_EVENTS_ENDPOINT`, `NEOSIGMA_ENABLED` (default true),
`NEOSIGMA_TRACING_ENABLED` (default false), `NEOSIGMA_ATTACH_TO_EXISTING_PROVIDER`
(default false), `NEOSIGMA_CONSOLE_EXPORT` (default false),
`NEOSIGMA_CAPTURE_CONTENT` (default true), `NEOSIGMA_MAX_CONTENT_CHARS`
(default 24000), plus OTel batch knobs `NEOSIGMA_MAX_QUEUE_SIZE`,
`NEOSIGMA_MAX_EXPORT_BATCH_SIZE`, `NEOSIGMA_SCHEDULE_DELAY_MILLIS`,
`NEOSIGMA_EXPORT_TIMEOUT_MILLIS`.

### Flush strategy by runtime shape (spans are lost without this)

Spans are batched and exported on a timer (default 5000 ms). A process that
exits before the timer fires loses every unflushed span. This is the single
most common reason a correct-looking integration produces no traces. Pick by
runtime shape:

- **Long-running server:** call `installShutdownHandlers()` once at startup. It
  registers SIGTERM/SIGINT handlers that flush spans AND events, then re-raise
  the signal so the process still exits. It coordinates across multiple loaded
  copies of the SDK (for example Next.js split server/instrumentation bundles),
  so every copy flushes on one signal. Keep the app's own HTTP-drain shutdown.
- **Serverless / per-invocation:** `await flush()` before returning from the
  handler. The background timer will not fire before the platform freezes the
  instance. `flush()` does not stop the SDK, so the next invocation still works.
- **CLI / script:** `await shutdown()` at the end.

`shutdown()` is terminal when the SDK owns the provider (the default): OpenTelemetry
allows one global provider per process, so a later `init()` cannot restore tracing.
Call `shutdown()` once at exit, never mid-run.

```ts
function flush(): Promise<void>;
function shutdown(): Promise<void>;
function installShutdownHandlers(): void;
function isInitialized(): boolean;
```

## 3. Open a turn per user message

`turn()` is the core primitive: one user message, one trace, one `turnId`.

```ts
function turn<T>(opts: TurnOptions, fn: (t: Turn) => T): T;
function startTurn(opts?: TurnOptions): Turn;

interface TurnOptions {
  sessionId?: string;   // the conversation; generated sess_... if omitted
  userMessage?: string; // captured as prompt content (privacy-gated)
  distinctId?: string;  // the end user
  agentId?: string;
  source?: string;      // default "api"
  sourceRef?: string;   // opaque back-reference into the app's system
  turnId?: string;      // generated turn_... if omitted; SUPPLY the app's own
                        // request/run id when one exists (join key for events)
}
```

`turn(opts, fn)` runs `fn` (sync or async), ends the turn when `fn` returns or
its promise settles, and records a thrown error as an ERROR-status span and
re-throws. It returns whatever `fn` returns.

```ts
import { turn } from "neosigma-sdk";

const answer = await turn(
  { sessionId: chatId, distinctId: userId, turnId: requestId, userMessage: text },
  async (t) => {
    const result = await runAgent(text); // model + tool spans nest here
    t.finish({ output: result.reply });  // record the output; see the pitfall below
    return result;
  },
);
```

**Pitfall: the callback return value is NOT recorded as the turn output.**
`turn(opts, fn)` passes `fn`'s return value straight back to the caller and does
not capture it as content (a handler usually returns a response object, not the
reply text). To record output, call `t.finish({ output })` before returning, or
use `turnHandler` (section 4e), whose `outputFrom` records the return value by
default.

The `Turn` handle (from the callback, or from `startTurn()`):

```ts
class Turn {
  readonly turnId: string;
  readonly sessionId: string;
  finish(opts?: { output?: string }): void;        // records output, ends span, idempotent
  setContent(content: { prompt?: string; completion?: string }): void;
  setAttributes(attrs: Record<string, AttributeValue>): void;
}
```

There is NO `setOutput`, `setInput`, or `end` method. Use `startTurn()` for a
lifecycle a single callback cannot wrap (open in one function, `finish()` in
another); guard the work between so `finish()` still runs, or the span leaks
unfinished.

Opening a `turn()` inside an active turn does not fork a second trace; it opens
a child span reusing the outer ids, so wrapping is safe.

`turn()` opens the root span only. Model and tool spans come from section 4. A
turn around uninstrumented work is a root span with no children.

## 4. Capture model and tool calls (pick per component)

Auto-instrumentation by library patching (`init({ tracingEnabled: true })`) is
NOT active in 0.5.0: the flag warns and does nothing. Use the adapters below,
which is the supported path today.

### 4a. Vercel AI SDK

```ts
import { registerTelemetry } from "ai";
import { OpenTelemetry } from "@ai-sdk/otel";
import { wrapAISDK } from "neosigma-sdk";
import * as ai from "ai";

registerTelemetry(new OpenTelemetry()); // REQUIRED once at startup on ai v7
const { generateText, streamText } = wrapAISDK(ai);
```

`wrapAISDK(ai)` returns wrapped `generateText`, `streamText`, `generateObject`,
and `streamObject` that enable the AI SDK's telemetry per call. Use them exactly
like the originals; each call emits `chat` / `invoke_agent` / `agent_step` spans
under the active turn, with token usage on canonical `gen_ai.usage.*` keys.

**Pitfall (ai v7): telemetry is a separate package.** As of ai v7, span emission
moved into `@ai-sdk/otel`. Without `registerTelemetry(new OpenTelemetry())` at
startup, the AI SDK emits ZERO spans and no warning, even though `wrapAISDK` ran
correctly. Install `@ai-sdk/otel` and call `registerTelemetry` once. `wrapAISDK`
enables the per-call flag but cannot perform the app-level registration.

Note: on ai v7 the same usage lands on both the leaf `chat` span and the
`invoke_agent` aggregate span, so a naive sum across a trace double-counts
tokens. This is a reporting characteristic, not a wrapper bug; do not strip it.

### 4b. LangChain

```ts
import { neosigmaCallbackHandler } from "neosigma-sdk";
import { turn } from "neosigma-sdk";

const handler = neosigmaCallbackHandler(); // reusable across invocations, incl. concurrent

await turn({ sessionId, distinctId, userMessage }, async () => {
  await chain.invoke(input, { callbacks: [handler] });
});
```

`neosigmaCallbackHandler()` returns a LangChain callback handler; pass it in the
`callbacks` array (per call, or on the model/chain constructor). It produces the
run tree (chain > llm > tool) as spans under the active turn, keyed by
LangChain's `runId`/`parentRunId`. One handler instance is safe to reuse and is
concurrency-safe (each run gets a fresh run id). Token usage is read from the
message's `usage_metadata`, so non-OpenAI providers (Anthropic and others) get
token counts, not only OpenAI.

Wrap the LangChain call in a `turn()` so the tree has a root and an id. Without
a surrounding turn the chain spans are orphaned.

### 4c. Claude Agent SDK

```ts
import { traceClaude, wrapClaudeQuery } from "neosigma-sdk";

for await (const message of traceClaude(query({ prompt }))) { ... }
// Or a reusable drop-in that wraps query():
const tracedQuery = wrapClaudeQuery();
```

`traceClaude` wraps the message stream from `query(...)`; iterate the wrapped
stream as usual. It rotates one turn per user message in the stream, with `chat`
and `execute_tool` children.

**Pitfall: a single-shot `query({ prompt })` opens no turn.** Turn rotation
triggers on user-type messages in the stream. A single-shot `query({ prompt })`
emits system, assistant, and result messages but no user message, so
`traceClaude` produces a lone `chat` span with no turn root and no `turnId`.
Wrap it in an explicit `turn()` yourself, or use the streaming-input form that
carries user messages:

```ts
await turn({ sessionId, distinctId, userMessage: prompt }, async () => {
  for await (const message of traceClaude(query({ prompt }))) { ... }
});
```

### 4d. Anthropic Managed Agents

```ts
import { wrapManagedAgents } from "neosigma-sdk";
const client = wrapManagedAgents(new Anthropic());
```

Use the wrapped client exactly as before (create session, stream events). The
adapter is a transparent proxy: every other method and property delegates
unchanged, and it emits one turn per user message with `chat`/`execute_tool`
children. No `turn()` needed; the adapter rotates turns itself.

### 4e. Tool functions and manual spans

```ts
import { tool, interaction, turnHandler } from "neosigma-sdk";

const search = tool(async (query: string) => db.search(query), { name: "search" });

const handleChat = turnHandler(
  async (req: ChatRequest) => llm.complete(req.message),
  {
    sessionFrom: (args) => (args[0] as ChatRequest).sessionId,
    messageFrom: (args) => (args[0] as ChatRequest).message,
    // outputFrom defaults to String(result), so the return value IS recorded here
  },
);
```

- `tool(fn, { name })` wraps a function as an `execute_tool` span (sets
  `gen_ai.tool.name`).
- `interaction(fn, { name })` wraps a function as an `invoke_agent` span but does
  NOT open a turn (no ids). Prefer `turn()` for the request boundary.
- `turnHandler(fn, { sessionFrom, messageFrom, outputFrom })` wraps a handler
  whose arguments carry the ids into a `turn()`. Unlike bare `turn()`, it records
  the return value as output by default.

**Pitfall: an inline anonymous arrow gets a generic span name.** `tool()` and
`interaction()` default the span name to the function's `.name`, which is empty
for an inline arrow (`tool(async (q) => ...)` produces a span literally named
`tool`). Pass `{ name }`, or wrap a named `function`.

## 5. Existing OpenTelemetry: dual export

Dual export sends every span to NeoSigma AND another OpenTelemetry backend at
once. There are two shapes; pick one.

**The SDK owns the provider (recommended), extra processors ride alongside.**
Pass the second backend as an `extraSpanProcessors` entry to `init()`:

```ts
import { init } from "neosigma-sdk";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";

init({
  apiKey: process.env.NEOSIGMA_API_KEY,
  extraSpanProcessors: [
    new BatchSpanProcessor(
      new OTLPTraceExporter({
        url: "https://api.smith.langchain.com/otel/v1/traces", // e.g. LangSmith
        headers: { "x-api-key": process.env.LANGSMITH_API_KEY, "Langsmith-Project": "my-project" },
      }),
    ),
  ],
});
// await shutdown() drains both legs; no separate flush of the extra processor needed.
```

**Attach to the app's existing provider.** If the app already builds and sets
its own provider, set `attachToExistingProvider: true` and call `init()` AFTER
the app's `setGlobalTracerProvider(...)`:

```ts
import { NeoSigmaSpanProcessor } from "neosigma-sdk";
const provider = new NodeTracerProvider({ spanProcessors: [new NeoSigmaSpanProcessor()] });
provider.register(); // app's own setup, FIRST
init({ attachToExistingProvider: true });
```

Key facts, several verified live:

- Install the exporter packages you reference: `@opentelemetry/exporter-trace-otlp-proto`
  (protobuf; NeoSigma ingest and most backends require protobuf, not JSON),
  `@opentelemetry/sdk-trace-base` (`BatchSpanProcessor`), and
  `@opentelemetry/sdk-trace-node` (`NodeTracerProvider`) for the attach shape.
- Requires OpenTelemetry JS 2.x. On 1.x the provider shape differs
  (`addSpanProcessor` exists) and attach behaves differently.
- OpenTelemetry JS 2.x providers take processors at construction and have no
  `addSpanProcessor`. Passing `attachToExistingProvider: true` against a 2.x
  provider logs a warning and does not attach. Construct `NeoSigmaSpanProcessor`
  and pass it to the provider at construction, as above.
- The second backend receives every span exactly as emitted. NeoSigma normalizes
  foreign attribute names (for example the AI SDK's) on its own export leg only,
  using a cloned view, so the other backend is unaffected regardless of which
  exporter flushes first.
- Any source works through dual export: native `turn()`/`tool()`, the AI SDK, and
  LangChain all fan out to both legs. AI SDK sources still need the section 4a
  `registerTelemetry` step, or both legs get zero AI SDK spans.

## 6. Concurrency and cross-process continuity

- Correlation rides `AsyncLocalStorage`: ids survive `await` and concurrent
  requests as long as each request has its own `turn()` or `trace()` scope.
  Concurrent turns via `Promise.all` produce distinct traces with no `turnId`
  cross-contamination.
- Ids do NOT cross a process, worker, or queue hop. Put `turnId`/`sessionId` in
  the job payload and re-bind on the far side: open a new `turn()` for a NEW
  user message (same `sessionId`, fresh turn id), or re-bind the SAME `turnId`
  with `trace()` for deferred work on an existing message. Each process runs its
  own `init()`/`shutdown()`.

## 7. Content and privacy

`init({ settings: { captureContent: false } })` (env
`NEOSIGMA_CAPTURE_CONTENT=false`) records metadata only (tokens, tool names,
timings) and drops prompt/completion/tool IO text everywhere, including adapters.
`maxContentChars` (default 24000; `0` disables truncation) truncates captured
text per field and appends a `... [truncated, N chars omitted]` marker.

## 8. Verify

1. Locally, set `NEOSIGMA_CONSOLE_EXPORT=true`, run one real request, and add
   `await flush()` (or `shutdown()`) before the process exits, or the console
   prints nothing. Confirm spans named `invoke_agent` (or `turn`), `chat`, and
   `execute_tool`, each carrying a `neosigma.turn_id` attribute. Console export
   without an API key proves shape, not delivery.
2. With `NEOSIGMA_API_KEY` set, run one request, then check the traces page at
   https://app.neosigma.ai: one trace per user message; an `invoke_agent` root;
   `chat` children with token usage; `execute_tool` children; the expected
   `sessionId`/`turnId`/`distinctId`.
3. Dual export: confirm the app's original backend still receives the same spans.

Troubleshooting:

| Symptom | Cause and fix |
| --- | --- |
| Nothing in NeoSigma, no errors | No `NEOSIGMA_API_KEY` in that environment, or `NEOSIGMA_ENABLED=false`. The SDK is silent by design; set the key. |
| Console export prints nothing on a short script | The process exited before the 5s batch flush. Add `await flush()` or `await shutdown()` before exit (section 2). |
| AI SDK calls produce zero spans | Missing `registerTelemetry(new OpenTelemetry())` from `@ai-sdk/otel` at startup (ai v7). `wrapAISDK` alone is not enough (section 4a). |
| Claude single-shot `query({ prompt })` has a `chat` span but no turn | A single-shot query emits no user message, so no turn opens. Wrap it in an explicit `turn()` (section 4c). |
| Span named `tool` / `interaction` instead of the function name | An inline anonymous arrow has no `.name`. Pass `{ name }` (section 4e). |
| `init({ tracingEnabled: true })` does nothing | Auto-instrumentation is a no-op in 0.5.0. Use the section 4 adapters. |
| Exports 404 | `otelEndpoint` was set to a base URL. It must be the full path ending `/v1/traces` (section 2). |
| Ingest returns 400 | A JSON OTLP exporter was used. Use `@opentelemetry/exporter-trace-otlp-proto` (protobuf). |
| Attach mode: warning, no dual export | `attachToExistingProvider: true` against an OTel 2.x provider, or `init()` ran before the app set its provider. Construct `NeoSigmaSpanProcessor` into the provider, or reorder (section 5). |
| Flat traces / spans missing a parent | The work is not inside an active turn (process/queue hop, or no `turn()`). Re-bind ids (section 6). |
| Spans stop partway through a run | Process exited without flushing. Wire the section 2 lifecycle for the runtime shape. |

## Mirror analytics events (optional)

If the app already sends PostHog or Mixpanel events, `wrapPosthog(client)` /
`wrapMixpanel(client)` return transparent proxies that mirror each event to
NeoSigma as a product event joined to the active turn, while the original
provider still receives it. This is product-event territory; see
[typescript-events.md](typescript-events.md) for `capture`/`identify` and the
turn-binding rules.
