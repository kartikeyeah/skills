# NeoSigma TypeScript product events integration

Instrument a TypeScript/Node codebase with the npm package `neosigma-sdk` so
product events (user actions) flow into NeoSigma and join the agent traces for
the same turn. This file covers the product-events surface of the TypeScript
SDK; for agent tracing (turns, the framework adapters, dual export) use
[typescript-tracing.md](typescript-tracing.md). Events and tracing run in one
process and share one id model. If something seems missing, check
https://docs.neosigma.ai/sdk/events rather than guessing.

## 1. Assess the codebase

- Where user actions happen: the request handlers, mutation endpoints, or UI
  backend that knows what the user did.
- Where the turn id lives: the id the tracing side uses for the same user
  message (often passed from the frontend or stored with the message row).
  The TypeScript `turnId` must equal the Python `turn_id` verbatim.
- Runtime shape: long-running server, serverless, or CLI (decides the flush
  strategy in section 4).
- Module system: the SDK is ESM-only and needs Node 18+. On CommonJS, load it
  inside an async context with `const neosigma = await import("neosigma-sdk")`
  (top-level `await` is not available in CommonJS).

## 2. Install and lifecycle

```bash
npm install neosigma-sdk
```

```ts
import { init, installShutdownHandlers } from "neosigma-sdk";

init(); // once at startup; reads NEOSIGMA_API_KEY; idempotent, never throws
installShutdownHandlers(); // once at startup; flushes the queue on SIGTERM/SIGINT
```

The product-events API surface (the tracing exports are in
[typescript-tracing.md](typescript-tracing.md)):

```ts
function init(overrides?: { apiKey?: string; eventsEndpoint?: string; enabled?: boolean }): void;
function trace<T>(ids: { turnId?: string; distinctId?: string; sessionId?: string }, fn: () => T): T;
function capture(eventName: string, properties?: Record<string, unknown>): void;
function identify(distinctId: string, properties?: Record<string, unknown>): void;
function flush(): Promise<void>;
function shutdown(): Promise<void>;
function installShutdownHandlers(): void;
```

Env vars: `NEOSIGMA_API_KEY` (required to send; without it every call is a
no-op), `NEOSIGMA_EVENTS_ENDPOINT` (default
`https://otel.neosigma.ai/v1/events`), `NEOSIGMA_ENABLED` (default true).

## 3. Bind the turn, emit events

Wrap each request in a `trace()` scope carrying the same `turnId` the agent
trace uses. Every `capture()` inside the scope, at any await depth, carries
the ids. `identify()` must be called inside a scope (outside one it is
dropped with a warning).

```ts
import { trace, capture, identify } from "neosigma-sdk";

app.post("/chat", async (req, res) => {
  await trace({ turnId: req.body.turnId, distinctId: req.user.id }, async () => {
    identify(req.user.id, { plan: req.user.plan });
    capture("message sent", { length: req.body.text.length });
    res.json(await handle(req));
  });
});
```

Rules that prevent silent data loss:

- Keys are camelCase (`turnId`, `distinctId`, `sessionId`). A snake_case key
  like `turn_id` is ignored without an error and the join is lost.
- `identify()` properties attach to the emitted `$identify` event only; they
  are not copied onto later events. Put per-event data in that event's own
  `properties`.
- Property values must be scalars (string, finite number, boolean). Others
  are coerced (Date to ISO string, objects JSON-stringified) or dropped.
- Do not emit event names starting with `$` (reserved).
- Concurrency is safe via AsyncLocalStorage as long as each request has its
  own `trace()` scope; concurrent sub-tasks identifying different users need
  their own scopes.
- Ids do not cross a process or queue hop; put `turnId` in the job payload
  and re-bind with `trace()` on the consumer.

## 4. Flush strategy by runtime shape

- **Long-running server:** `installShutdownHandlers()` at startup covers
  deploys/scale-in (flushes, then re-raises the signal). Keep the app's own
  HTTP-drain shutdown logic; the handler only flushes the SDK queue.
- **Serverless:** the background flush timer may never fire before the
  invocation freezes. `await flush()` before returning from the handler.
- **CLI/script:** `await shutdown()` at the end.

Events are batched in memory and sent in the background; `capture()` never
blocks or throws. A 401/403 (revoked key) disables sending for the process.
Queue overflow drops new events with a rate-limited warning.

## 5. Verify

With `NEOSIGMA_API_KEY` set, trigger one instrumented action, then confirm
the event appears in NeoSigma alongside the turn's trace (same `turn_id`).
If events show but do not join a trace, compare the exact `turnId` string
against the Python side's `turn_id` for that message; the join is verbatim
string equality.
