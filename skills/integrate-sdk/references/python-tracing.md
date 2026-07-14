# NeoSigma Python tracing integration

Instrument a Python codebase with `neosigma-sdk` so agent runs land in
NeoSigma as traces. This file is the API surface of record for the Python SDK
(0.4.x). Parameters not listed here do not exist.

## 1. Assess the codebase

Find, in this order:

- The real execution path: the route, job, queue consumer, or CLI command that
  runs the agent.
- The LLM surface: Anthropic Managed Agents, Claude Agent SDK, raw
  `anthropic`/`openai` clients, or something else.
- Existing OpenTelemetry: any `TracerProvider` construction or
  `trace.set_tracer_provider(...)` call. This decides the dual-export
  handling in section 5.
- The app's own ids: conversation/chat id, end-user id, and any per-request or
  per-run row id (these become `session_id`, `distinct_id`, `turn_id`).

Useful search:

```bash
rg -n "anthropic|openai|claude_agent_sdk|set_tracer_provider|TracerProvider|FastAPI|Starlette" .
```

## 2. Install and lifecycle

```bash
pip install neosigma-sdk                      # or: uv add neosigma-sdk
pip install "neosigma-sdk[instrumentation]"   # only if auto-instrumenting raw LLM clients
pip install "neosigma-sdk[fastapi]"           # only if using the FastAPI middleware
```

```python
import neosigma_sdk as neosigma

neosigma.init()        # once at startup; reads NEOSIGMA_API_KEY from the env
...
neosigma.shutdown()    # once at process exit; flushes buffered spans
```

The exact `init` signature (no other keyword arguments exist; there is NO
`tracer_provider`, `instrument`, or `endpoint` argument):

```python
def init(
    api_key: str | None = None,
    *,
    project: str | None = None,
    tracing_enabled: bool | None = None,           # auto-instrumentation flag ONLY
    attach_to_existing_provider: bool | None = None,  # dual export, section 5
    settings: Settings | None = None,              # full config object, fields below
) -> None
```

- `init()` is idempotent and fail-open. It never raises into app startup; on
  any failure it logs and stays disabled.
- Without an API key every SDK call is a no-op, so the integration is safe to
  merge before keys are provisioned.
- Wire `shutdown()` into process exit: `atexit.register(neosigma.shutdown)`,
  or a FastAPI `lifespan` handler, or the worker's finally block. In a
  long-running server call it once on graceful stop, never per request.
  `neosigma.flush()` force-flushes without stopping. Note: `atexit` and
  `finally` do not run on an unhandled SIGTERM; in workers and containers,
  convert SIGTERM to a clean exit (e.g.
  `signal.signal(signal.SIGTERM, lambda *_: sys.exit(0))`) so the flush runs.
- `shutdown()` is terminal when the SDK owns the provider; call it at exit,
  not mid-run.

Settings fields / env vars (env prefix `NEOSIGMA_`): `api_key`, `project`
(default `default`), `service_name` (defaults to project), `otel_endpoint`
(default `https://otel.neosigma.ai/v1/traces`), `events_endpoint`
(default `https://otel.neosigma.ai/v1/events`), `enabled` (default true),
`tracing_enabled` (default false), `attach_to_existing_provider` (default
false, env `NEOSIGMA_ATTACH_TO_EXISTING_PROVIDER`), `capture_content`
(default true),
`max_content_chars` (default 24000), `console_export` (default false), plus
OTel batch knobs `max_queue_size`/`max_export_batch_size`/
`schedule_delay_millis`/`export_timeout_millis`.

## 3. Open a turn per user message

`turn()` is the core primitive: one user message, one trace, one `turn_id`.
Exact signature (keyword-only; there is NO `name`, `input`, `user_id`, or
`links` parameter):

```python
def turn(
    *,
    session_id: str = "",      # the conversation; generated sess_... if omitted
    user_message: str | None = None,
    distinct_id: str = "",     # the end user
    agent_id: str = "",
    source: str = "api",
    source_ref: str = "",      # opaque back-reference into the app's system
    turn_id: str = "",         # generated turn_... if omitted; SUPPLY the app's
                               # own request/run id when one exists (join key
                               # for product events)
) -> Turn
```

The `Turn` handle: `finish(output=None)` (records output, sets OK, ends span,
idempotent), `set_content(prompt=None, completion=None)`,
`set_attributes(mapping)`, and attributes `.turn_id` / `.session_id`. There is
NO `set_output`, `set_input`, or `end` method.

Prefer the context-manager form: it always finishes the turn (including when
the work raises) and records exceptions as errors, which the manual form does
not. Record output via `set_content` inside the block:

```python
# Handler shape: map the app's ids onto the turn.
with neosigma.turn(
    session_id=chat_id,
    distinct_id=user_id,
    turn_id=request_id,        # app's own id when it has one
    user_message=message_text,
) as t:
    result = run_agent(message_text)
    t.set_content(completion=result.output)
```

The manual form (`t = neosigma.turn(...)` then `t.finish(output=...)`) exists
for lifecycles a single block cannot wrap; if the work in between can raise,
guard it so `finish()` still runs, or the turn span leaks unfinished.

Opening a `turn()` inside an active turn does not fork a second trace; it
opens a child span reusing the outer ids, so wrapping is safe.

`turn()` opens the root span only. Model and tool spans come from sections 4a
to 4d.

## 4. Capture model and tool calls (pick per component)

### 4a. Anthropic Managed Agents

```python
client = neosigma.wrap_managed_agents(anthropic.Anthropic())  # or AsyncAnthropic
```

Use the wrapped client exactly as before (create session, stream events). The
adapter emits one turn per user message with `chat` and `execute_tool`
children, token usage included. No `turn()` needed around it; the adapter
rotates turns itself.

### 4b. Claude Agent SDK

```python
# For query():
async for message in neosigma.trace_claude(claude_agent_sdk.query(prompt=p)):
    ...
# Or a reusable drop-in:
traced_query = neosigma.wrap_claude_query()

# For the stateful ClaudeSDKClient (patches receive_response once):
neosigma.ClaudeTracingProcessor().configure()
```

`query()` is traced ONLY via these wrappers. A `from claude_agent_sdk import
query` binding is never patched; do not attempt to monkeypatch it.

### 4c. Raw Anthropic / OpenAI clients

```python
neosigma.init(tracing_enabled=True)   # requires the [instrumentation] extra
```

Client calls (`client.messages.create`, chat completions) become `chat` spans
under whichever turn is active. Instrumentation patches the library, not
individual clients, so clients constructed before `init()` are captured too.
`tracing_enabled` controls ONLY this auto-instrumentation; turns, decorators,
and adapters trace regardless.

### 4d. Tool functions and everything else

```python
@neosigma.tool()                 # execute_tool span per call; name defaults to
def search_kb(query: str): ...   # the function name, override: @neosigma.tool("kb")
```

`@neosigma.interaction()` traces a function as an `invoke_agent` root span but
does NOT create a turn (no ids); prefer `turn()` for the request boundary.
`@neosigma.turn_handler(session_from=..., message_from=..., output_from=...)`
wraps a handler whose arguments already carry the ids.

### 4e. FastAPI / Starlette

```python
from neosigma_sdk.integrations.fastapi import NeosigmaTurnMiddleware
app.add_middleware(NeosigmaTurnMiddleware)   # requires the [fastapi] extra
```

Opens a turn per request: `session_id` from the `x-neosigma-session-id`
request header, `distinct_id` from `x-neosigma-distinct-id`; returns the turn
id in the `x-neosigma-turn-id` response header. Optional kwargs: an async
`message_extractor(request)` for the user message and a `path_filter(path)`
to skip routes. It finishes turns without an output; use an explicit `turn()`
or `@turn_handler` in the handler instead when the response must be captured.

## 5. Existing OpenTelemetry: dual export

Provider posture is EXPLICIT, controlled by `attach_to_existing_provider`
(init kwarg or env `NEOSIGMA_ATTACH_TO_EXISTING_PROVIDER`, default false).
Every mismatch logs a warning and stays disabled rather than guessing:

- **Default (`False`):** if no real global provider exists, the SDK creates
  its own `TracerProvider` and installs it as the OTel global. If a real
  provider is ALREADY set, the SDK does not touch it, warns ("a
  TracerProvider is already configured, so NeoSigma is not attaching and
  tracing stays dark"), and stays disabled.
- **Attach (`True`):** the SDK adds its processors to the app's existing
  provider (dual export). The app's own exporters keep receiving every span,
  and NeoSigma additionally receives every span emitted through that
  provider. Requires the provider to exist first: with the flag set and no
  provider configured, the SDK warns and stays disabled. A provider without
  a callable `add_span_processor` also warns and stays disabled.

So for an app with existing OpenTelemetry, dual export takes BOTH the flag
and the ordering (app's provider first):

```python
provider = TracerProvider(resource=...)
provider.add_span_processor(BatchSpanProcessor(their_exporter))
trace.set_tracer_provider(provider)   # app's setup, unchanged, FIRST

neosigma.init(attach_to_existing_provider=True)   # dual export
```

The alternative posture also works: call `neosigma.init()` FIRST (no flag) so
the SDK owns the global provider, then add the app's other exporter to that
global provider. Pick one; do not mix.

In attached mode `shutdown()`/`flush()` touch only NeoSigma's own processors;
the app's provider and exporters are left alone. The app's own HTTP/infra
spans also reach NeoSigma in this mode; that is expected and benign.

## 6. Cross-process and worker continuity

Correlation ids ride contextvars: they survive `await` but not a process,
thread-pool, or queue hop. Thread the ids through the job payload and re-bind
on the far side; do not rely on OTel context propagation for turn identity.
Choosing what to re-bind: a job that handles a NEW user message opens a new
`turn()` (same `session_id`, fresh turn id); deferred work for an EXISTING
message re-binds the SAME `turn_id` via `trace()` or `turn(turn_id=...)`.

```python
# Producer
enqueue(job, turn_id=t.turn_id, session_id=t.session_id, user_id=user_id)

# Consumer (separate process): same session, new or same turn
with neosigma.turn(session_id=job.session_id, distinct_id=job.user_id,
                   user_message=job.message):
    handle(job)
# Or, when spans are produced by an adapter and only ids must bind:
with neosigma.trace(turn_id=job.turn_id, session_id=job.session_id):
    handle(job)
```

Each process runs its own `init()`/`shutdown()`.

## 7. Content and privacy

`capture_content=False` (env `NEOSIGMA_CAPTURE_CONTENT=false`) records
metadata only (tokens, tool names, timings) and drops prompt/completion/tool
IO text everywhere, including adapters and auto-instrumentation.
`max_content_chars` (default 24000) truncates captured text per field.

## 8. Verify

1. Locally, set `NEOSIGMA_CONSOLE_EXPORT=true` and run one real request:
   spans print to stdout. Console export without an API key logs a warning
   that nothing reaches NeoSigma; it proves shape, not delivery.
2. With `NEOSIGMA_API_KEY` set, run one request, then check the traces page
   at https://app.neosigma.ai. Confirm: one trace per user message; an
   `invoke_agent` root; `chat` children carrying token usage; `execute_tool`
   children for tools; the expected `session_id`/`turn_id`/`distinct_id`.
3. Dual export: also confirm the app's original backend still receives spans.

Troubleshooting:

| Symptom | Cause and fix |
| --- | --- |
| Nothing in NeoSigma, no errors | No `NEOSIGMA_API_KEY` in that environment, or `NEOSIGMA_ENABLED=false`. The SDK is silent by design; set the key. |
| Console spans print but platform is empty | Console export is on without a key (the SDK logs exactly this warning). Set the key. |
| Log: "a TracerProvider is already configured ... tracing stays dark" | The app has its own OTel and the attach flag is off. Pass `attach_to_existing_provider=True` (section 5). |
| Log: "attach_to_existing_provider=True but no TracerProvider is configured" | The flag is set but `init()` ran before the app's `set_tracer_provider`. Move `init()` after it, or drop the flag to let the SDK own the provider. |
| Log: "no callable add_span_processor" | The installed provider is not a standard SDK `TracerProvider`. The SDK stays disabled; use one that accepts span processors. |
| Flat traces / spans missing a parent | The work is not running inside an active turn (thread/process hop, or no `turn()` opened). Re-bind ids per section 6. |
| `query()` calls produce no spans | The stream was not wrapped. Wrap with `trace_claude`/`wrap_claude_query`; imports of `query` are never patched. |
| Spans stop after some point in a run | Process exited without `shutdown()`; buffered spans were dropped. Wire section 2's lifecycle. |
