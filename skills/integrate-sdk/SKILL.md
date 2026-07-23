---
name: integrate-sdk
description: Integrates the NeoSigma SDKs into a codebase so traces and product events flow into NeoSigma. Activates on requests like "integrate NeoSigma", "add NeoSigma tracing to this app", "instrument this repo with neosigma-sdk", "get our traces into NeoSigma", "onboard this codebase to NeoSigma", "set up the NeoSigma SDK", "send product events to NeoSigma", "wire NeoSigma into our existing OpenTelemetry setup", or "dual export to NeoSigma and another backend". Do NOT use for analyzing traces already in NeoSigma, building evals, or developing the NeoSigma platform itself.
---

# Integrate the NeoSigma SDK

When activated, instrument the attached codebase with the NeoSigma SDKs so
every agent run lands in NeoSigma as a trace and, where the app emits them,
product events join those traces. Work from the reference files in this skill,
not from memory: the SDK surface is small and exact, and plausible-looking
parameters that are not in the references do not exist. The output is working
instrumentation code in the codebase plus a verification that spans
reached NeoSigma.

## Core principles

1. **References first, memory never.** Read the matching reference file before
   writing any SDK call. Do not invent parameters, wrappers, or attribute
   names beyond what the references state. If something is missing, stop rather
   than guessing.
2. **Instrument the real path.** Find the route, job, queue consumer, or CLI
   command that actually runs the agent, and instrument that. Do not build a
   parallel demo.
3. **One turn per user message.** NeoSigma's unit is the turn: one user
   message plus everything the agent did in response, one trace, one
   `turn_id`. Map the app's own conversation id to `session_id`, its end-user
   id to `distinct_id`, and reuse its own request/run id as `turn_id` only when
   that id is one-to-one with a user message.
4. **Prefer supported integrations over manual spans.** Managed Agents, the
   Claude Agent SDK, and raw Anthropic/OpenAI clients have dedicated capture
   paths. FastAPI middleware opens the turn; a provider adapter or
   auto-instrumentor captures model calls, and `tool()` captures remaining
   tools.
5. **Never break the existing telemetry.** Both SDKs build their own private
   provider by default, so NeoSigma runs alongside an existing OpenTelemetry
   setup without changing it and without capturing its spans. Neither SDK has an
   attach option. For dual export of the agent trace to the other backend too,
   in TypeScript pass the other backend's processor through `extraSpanProcessors`
   (or use the `tracerProvider` handoff); in Python use the `tracer_provider`
   handoff per that reference. Either way, do not replace or reorder the app's
   own OpenTelemetry setup, and verify both destinations.
6. **Keys stay out of chat and out of code.** Ask the user to set
   `NEOSIGMA_API_KEY` (an `ns_live_...` key from Settings > Developer > API
   Keys) in their environment. Never paste a key into a file or a message.
   Without a key there is no network export, so merged instrumentation is safe
   in environments that lack one. Console span export remains available for
   local shape verification.
7. **Verify with evidence, not code review.** Finish by running one real
   request and confirming the trace in NeoSigma (or spans on stdout via
   console export locally). For Next.js, also run `next build` after the
   integration and keep the SDK out of client and Edge code. Instrumentation
   that has not produced a visible trace is not done.

## Workflow

0. **Route by language and surface.** Both SDKs do full agent tracing; the
   TypeScript workflow in this skill also covers product events. Agent traces
   from Python: use
   [references/python-tracing.md](references/python-tracing.md). Agent traces
   from TypeScript/Node: use
   [references/typescript-tracing.md](references/typescript-tracing.md).
   Product events from TypeScript: use
   [references/typescript-events.md](references/typescript-events.md). A
   full-stack app usually needs the tracing reference for its agent's language
   plus the TypeScript events reference, correlated per principle 3. If the
   agent code is in neither language, say so and stop; do not improvise an SDK.
1. **Assess the codebase** per the matching reference's section 1 (execution
   path, LLM surface, existing OpenTelemetry, the app's own ids).
2. **Instrument** following the reference exactly: preserve any existing
   OpenTelemetry provider first, then add SDK lifecycle, the capture surface
   per component, and id correlation.
3. **Run one real request and verify** per the reference's verification
   section. Fix until the trace (and events, if any) are visible.
4. **Report** using the output format below.

## Use case references

- Python tracing (turns, adapters, auto-instrumentation, FastAPI middleware,
  the private-default provider and dual export via the `tracer_provider` handoff,
  lifecycle, verification):
  [references/python-tracing.md](references/python-tracing.md)
- TypeScript tracing (turns, the Vercel AI SDK / LangChain / Claude Agent SDK /
  Managed Agents adapters, the private-default provider and dual export via
  `extraSpanProcessors`, flush-by-runtime-shape, verification):
  [references/typescript-tracing.md](references/typescript-tracing.md)
- TypeScript product events (capture/identify, binding turns, concurrency,
  serverless, shutdown): [references/typescript-events.md](references/typescript-events.md)
- Correlating across the SDKs: the references share the id model in principle 3.
  A `turnId`/`turn_id` for the same user message must match verbatim across
  languages; ids are camelCase in TypeScript code and snake_case on the wire.

## Output format

Deliver: (1) the instrumentation diff, following the repo's existing code
style; (2) a short summary listing which surface was used per
component (adapter, flag, middleware, or manual turns) and the id mapping
chosen (`session_id`/`turn_id`/`distinct_id` sources); (3) the verification
result: what was run and where the trace was observed. Do NOT include: API
keys, invented SDK parameters, speculative claims about SDK behavior not in
the references, or instrumentation for code paths the app does not execute.

## Examples

**Happy path.** "Add NeoSigma tracing to our FastAPI support bot" with a raw
Anthropic client and two tool functions. Read the Python reference. Install
the `[instrumentation]` extra, wire `neosigma.init(tracing_enabled=True)` at
startup and `shutdown()` on exit,
open one `turn()` per request in the chat handler (session/user/message ids
from the request), decorate the tool functions with `@neosigma.tool()`, run
one request with console export, then confirm the trace in NeoSigma.

**Happy path (TypeScript).** "Add NeoSigma tracing to our Node chatbot" that
uses LangChain, with the Vercel AI SDK in one path. Read the TypeScript tracing
reference. Call `init()` and `installShutdownHandlers()` at startup (long-running
server), open a `turn()` per request in the chat route (session/user/message ids
from the request), and pass `neosigmaCallbackHandler()` in the LangChain
`callbacks`. For the AI SDK path, install `@ai-sdk/otel` and use `wrapAISDK(ai)`,
or that path emits no spans. Run one request with `NEOSIGMA_CONSOLE_EXPORT=true` plus `await flush()`
to see the spans, then confirm the trace in NeoSigma.

**Edge case: existing OpenTelemetry.** The app already installs its own
TracerProvider exporting to another backend. Both SDKs build their own private
provider by default and coexist with it automatically, no flag, without
capturing its spans, so call `init()` as usual. There is no attach option
(removed in both SDKs). For dual export of the agent trace to that backend too,
in TypeScript pass `extraSpanProcessors` (or the `tracerProvider` handoff); in
Python use the `tracer_provider` handoff, building one provider with
`CorrelationSpanProcessor` before `NeoSigmaSpanProcessor` plus your exporter.
Verify both destinations receive data.

**Edge case: no key available in the session.** The user has not provisioned
an API key. Complete the instrumentation anyway (nothing is sent to NeoSigma
without a key), verify span shape locally with
`NEOSIGMA_CONSOLE_EXPORT=true`, and tell the user which env var to set in each
environment to turn exporting on.
