---
name: import-verifiers
description: Imports a codebase's existing quality checks into NeoSigma as LLM-as-judge verifiers, using the NeoSigma MCP. Activates on requests like "import our verifiers into NeoSigma", "set up verifiers from my codebase", "add our evals to NeoSigma", "onboard our judge checks to NeoSigma", or "turn our eval suite into NeoSigma verifiers". Requires the NeoSigma MCP to be connected. Do NOT use for creating a single verifier by hand (call create_verifier directly), for analyzing traces already in NeoSigma, or for integrating the tracing SDK (use integrate-sdk).
---

# Import verifiers into NeoSigma

When activated, find the checks in the attached codebase that judge the quality
or correctness of an agent's output, translate each into a self-contained
NeoSigma verifier, and create it through the NeoSigma MCP. A NeoSigma verifier
is an LLM-as-judge: a natural-language criterion evaluated against one
production trace, with no access to code or runtime state. This skill uses the
MCP tools `list_verifiers` and `create_verifier`, so the NeoSigma MCP must be
connected (see the NeoSigma docs for connecting it).

## Core principles

1. **Only what a trace-only judge can evaluate.** A verifier sees one production
   trace plus its own instructions, nothing else. Import a check only if its
   intent can be judged from the trace alone. Skip mechanical checks the judge
   cannot reproduce: exact-value assertions, latency thresholds, status codes,
   and checks on internal variables or on external state absent from the trace.
2. **Translate intent, never code.** Read each check, capture what it is really
   testing, and write it as a standalone criterion. Do not paste, reference, or
   paraphrase source code, file names, functions, or variables. The judge never
   sees them.
3. **State the pass and the fail.** Each verifier's instructions say what to
   check, what a passing trace looks like, and what makes a trace fail. Keep it
   to one criterion per verifier.
4. **Deduplicate first.** Call `list_verifiers` before creating anything and
   skip checks already covered by an existing verifier.
5. **Report what you did.** After creating verifiers, tell the user which checks
   became verifiers and which you skipped and why, so they can review before the
   verifiers score traffic.

## Workflow

1. **Find the checks.** Search the codebase for anything that judges agent or
   LLM output quality or correctness: LLM-as-judge / model-graded evals,
   scorers, graders, rubric prompts, and assertions whose intent is qualitative
   (for example "the reply must not leak PII"). Common homes are an `evals/`,
   `evaluations/`, or `tests/` directory, or files named `*eval*`, `*judge*`,
   `*scorer*`, or `*grader*`.
2. **Classify each check.** Decide import (a trace-only judge can evaluate its
   intent) or skip (mechanical, or needs code execution or state not in the
   trace). Record a one-line reason for every skip.
3. **Deduplicate.** Call `list_verifiers` and drop any check already covered.
4. **Translate and create.** For each check to import, write a short kebab-case
   `judge` handle and self-contained `instructions` (what to check, passing
   trace, failing trace, no code references), then call `create_verifier` once
   per verifier.
5. **Report** using the output format below.

## Output format

Deliver a short summary: (1) the verifiers created, each as its `judge` handle
plus a one-line description; (2) the checks skipped, each with a one-line
reason; (3) any check you were unsure about, flagged for the user to confirm. Do
NOT paste source code into verifier instructions, create a verifier for a check
a trace-only judge cannot evaluate, or invent NeoSigma tool parameters beyond
`create_verifier`'s (`judge`, `instructions`, `description`, `enabled`).

## Examples

**Happy path.** "Import our support agent's evals into NeoSigma." The repo has
`evals/quality.py` with three LLM-judge functions (answers-the-question,
grounded-in-context, concise) and `evals/safety.py` with a PII-leak regex check.
Read them, call `list_verifiers` (empty), then create four verifiers: the three
quality judges and a `no-pii-leak` verifier ("the reply must not expose a
customer's unmasked email or phone number; a masked value is fine"). Report the
four created.

**Edge case: mixed with unit tests.** The repo also has `tests/test_pipeline.py`
asserting response ids, latency under 500ms, and cache size. Skip all three: an
exact id, a latency budget, and an internal cache count cannot be judged from a
trace. Report them as skipped with reasons.

**Edge case: depends on external state.** A refund-policy check takes a
`manager_approved` boolean the trace does not contain. Skip it, or, if the
approval is visible in the trace, write the verifier to key off that visible
signal. Flag it for the user either way.
