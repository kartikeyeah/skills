---
name: integrate-sdk
description: Integrates the NeoSigma SDKs into a codebase so traces and product events flow into NeoSigma. Activates on requests like "integrate NeoSigma", "add NeoSigma tracing to this app", "instrument this repo with neosigma-sdk", "get our traces into NeoSigma", "onboard this codebase to NeoSigma", "set up the NeoSigma SDK", "send product events to NeoSigma", "wire NeoSigma into our existing OpenTelemetry setup", or "dual export to NeoSigma and <backend>". Do NOT use for analyzing traces already in NeoSigma, building evals, or developing the NeoSigma platform itself.
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
   names beyond what the references state. If something seems missing, check
   https://docs.neosigma.ai rather than guessing.
2. **Instrument the real path.** Find the route, job, queue consumer, or CLI
   command that actually runs the agent, and instrument that. Do not build a
   parallel demo.
3. **One turn per user message.** NeoSigma's unit is the turn: one user
   message plus everything the agent did in response, one trace, one
   `turn_id`. Map the app's own conversation id to `session_id`, its end-user
   id to `distinct_id`, and reuse its own request/run id as `turn_id` when one
   exists.
4. **Prefer adapters and auto-instrumentation over manual spans.** If the app
   uses Anthropic Managed Agents, the Claude Agent SDK, FastAPI, or raw
   Anthropic/OpenAI clients, a wrapper or flag captures model and tool calls
   with no per-call code. Reach for manual `@neosigma.tool()` spans only for
   what remains.
5. **Never break the existing telemetry.** If the app already configures
   OpenTelemetry, dual export is an explicit opt-in:
   `neosigma.init(attach_to_existing_provider=True)`, called AFTER the app's
   own provider setup. Without the flag the SDK stays dark next to an
   existing provider (it warns, it never stomps). Follow the dual-export
   section of the Python reference exactly.
6. **Keys stay out of chat and out of code.** Ask the user to set
   `NEOSIGMA_API_KEY` (an `ns_live_...` key from Settings > Developer > API
   Keys) in their environment. Never paste a key into a file or a message.
   Without a key the SDKs are no-ops, so merged instrumentation is safe in
   environments that lack one.
7. **Verify with evidence, not code review.** Finish by running one real
   request and confirming the trace in NeoSigma (or spans on stdout via
   console export locally). Instrumentation that has not produced a visible
   trace is not done.

## Workflow

0. **Route by what must flow.** Agent traces from Python code: use
   [references/python-tracing.md](references/python-tracing.md). Product
   events from TypeScript code: use
   [references/typescript-events.md](references/typescript-events.md). A
   full-stack app usually needs both, correlated per principle 3. If the
   agent code is in neither language, say so and stop; do not improvise an
   SDK.
1. **Assess the codebase** per the matching reference's section 1 (execution
   path, LLM surface, existing OpenTelemetry, the app's own ids).
2. **Instrument** following the reference exactly: lifecycle first
   (`init`/`shutdown`), then the capture surface per component, then id
   correlation.
3. **Run one real request and verify** per the reference's verification
   section. Fix until the trace (and events, if any) are visible.
4. **Report** using the output format below.

## Use case references

- Python tracing (turns, adapters, auto-instrumentation, FastAPI middleware,
  dual export with an existing TracerProvider, lifecycle, verification):
  [references/python-tracing.md](references/python-tracing.md)
- TypeScript product events (capture/identify, binding turns, concurrency,
  serverless, shutdown): [references/typescript-events.md](references/typescript-events.md)
- Correlating the two: both references share the id model in principle 3. The
  TypeScript `turnId` must equal the Python `turn_id` for the same user
  message; ids are camelCase in TypeScript code and snake_case on the wire.

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

**Edge case: existing OpenTelemetry.** The app already installs its own
TracerProvider exporting to another backend. Do not replace it and do not
reorder its setup. Call `neosigma.init(attach_to_existing_provider=True)`
AFTER the app's `trace.set_tracer_provider(...)` so the SDK attaches to that
provider (dual export: both backends keep receiving spans). Without the flag
the SDK warns and stays dark next to an existing provider. Verify both
destinations still receive data.

**Edge case: no key available in the session.** The user has not provisioned
an API key. Complete the instrumentation anyway (the SDK is a no-op without a
key, so nothing breaks), verify shape locally with
`NEOSIGMA_CONSOLE_EXPORT=true`, and tell the user which env var to set in each
environment to turn exporting on.
