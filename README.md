# NeoSigma Skills

[Agent Skills](https://github.com/anthropics/skills) that teach AI coding assistants how to work with [NeoSigma](https://neosigma.ai): instrument a codebase with the NeoSigma SDK so agent runs land as traces and product events join them.

## Skills

| Skill                                   | Description                                                                                                                                                                       |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [integrate-sdk](./skills/integrate-sdk) | Integrate the NeoSigma SDK: agent tracing (Python), product events (TypeScript), auto-instrumentation, and dual export with an existing OpenTelemetry setup, correlated per turn. |

## Installation

### Use your coding agent

Use your coding agent with this instruction so it can install the NeoSigma
skill and apply it to your task.

```txt
Install the integrate-sdk skill from github.com/neosigmaai/skills
and use it to add NeoSigma tracing to this application
following NeoSigma best practices.
```

### Cursor

Install as a [Cursor plugin](https://cursor.com/docs/plugins):

```
/add-plugin neosigma
```

Or via the skills CLI:

```bash
npx skills add neosigmaai/skills --skill "integrate-sdk" --agent cursor
```

### Claude Code

Add the marketplace and install:

```bash
claude plugin marketplace add neosigmaai/skills
claude plugin install neosigma@neosigma-skills
```

Or via the skills CLI:

```bash
npx skills add neosigmaai/skills --skill "integrate-sdk" --agent claude-code
```

### Install with npx

```bash
npx skills add neosigmaai/skills --skill "integrate-sdk"
```

## Prerequisites

Set your NeoSigma API key before asking an agent to instrument your codebase. Generate an `ns_live_...` key in Settings > Developer > API Keys.

```bash
export NEOSIGMA_API_KEY=...
```

Without a key the SDK is a no-op, so instrumentation is safe to merge before keys are provisioned.

## Usage

Once installed, your agent can use this skill when you ask it to:

- Add NeoSigma tracing to a Python agent or workflow
- Send product events from a TypeScript/Node app, correlated to the agent trace by turn
- Auto-instrument raw Anthropic/OpenAI clients, the Claude Agent SDK, or Anthropic Managed Agents
- Add a FastAPI/Starlette middleware that opens a turn per request
- Dual-export to NeoSigma alongside an existing OpenTelemetry backend
- Verify that traces and events reached NeoSigma
