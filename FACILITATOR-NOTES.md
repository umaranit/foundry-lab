# Facilitator Notes — Invoice Discrepancy Assistant Lab

> **Status: skeleton only.** This file is a placeholder structure for facilitator guidance. Each
> section below needs to be filled in during the full-content pass. Do not use this as-is to run
> a live workshop yet.

## How to use this document

TODO: Add guidance on how a facilitator should use this document before, during, and after the
workshop (e.g., pre-reading, pre-provisioning shared resources, timing cues to give participants).

## Overall timing guidance

TODO: Add a timing table/run-of-show for the full ~5.75-hour session, including suggested break
points (e.g., after Module 4 and after Module 8), buffer time for Azure provisioning delays,
guidance on which modules can be trimmed if the group is running behind, and notes on splitting
the workshop across two half-day sessions (e.g., Modules 0–5 on day one, Modules 6–10 on day
two).

| Module | Planned time | Buffer notes |
|---|---|---|
| 0 — Prerequisites | 15 min | TODO |
| 1 — Foundry project & model setup | 20 min | TODO |
| 2 — Build the agent (portal) | 25 min | TODO |
| 3 — Knowledge base / grounding | 30 min | TODO |
| 4 — Tools — Built-in (Portal) | 15 min | TODO |
| 5 — Custom Function Tools (Code) | 45 min | TODO |
| 6 — MCP Integration (Code) | 45 min | TODO |
| 7 — Multi-agent orchestration (Agent Framework) | 50 min | TODO |
| 8 — Evaluation | 45 min | TODO |
| 9 — Monitoring & tracing | 40 min | TODO |
| 10 — Wrap-up & governance | 15 min | TODO |

## Module 0 — Prerequisites

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., subscription access/quota approvals not requested far enough in advance)

### Talking points
TODO

## Module 1 — Foundry project & model setup

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., model/region availability, quota limits)

### Talking points
TODO

## Module 2 — Build the agent (portal)

### Timing guidance
TODO

### Common pitfalls
TODO

### Talking points
TODO

## Module 3 — Knowledge base / grounding

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., indexing delays, citation formatting confusion)

### Talking points
TODO

## Module 4 — Tools — Built-in (Portal)

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., Code Interpreter file-upload issues; participants expecting function/MCP tools here —
clarify those are Modules 5 and 6)

### Talking points
TODO

## Module 5 — Custom Function Tools (Code)

### Timing guidance
TODO

### Common pitfalls
TODO — **especially:** participants using the legacy threads/runs API instead of
`openai_client.conversations.create()` / `openai_client.responses.create()`; forgetting to call
`project_client.agents.create_version` after changing the tool definition so the hosted agent
doesn't pick up the new `FunctionTool`; function tool schema mismatches.

### Talking points
TODO — clarify this still calls a server-hosted agent (now at a new version), it does not define
a new agent framework/runtime; contrast with Module 7.

## Module 6 — MCP Integration (Code)

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., participants mixing sync and async code or forgetting `asyncio.run(main())` during the
Module 5→6 sync-to-async client transition; the FastMCP server subprocess not starting or not
being found because `client.py` was launched from a different terminal/venv than the one with
`fastmcp`/`mcp` installed; escalation contact data not matching the policy's Section 8 table
structure)

### Talking points
TODO — reinforce that this is another new agent version (same pattern as Module 5); the MCP
server's tool is still wrapped as a plain `FunctionTool`, same as Module 5's PO lookup. There's no
`MCPTool` class or approval step for this **local** MCP server (that pattern is for remote MCP
servers only) — the MCP server runs locally and is grounded in existing policy content rather
than new content.

## Module 7 — Multi-agent orchestration with Microsoft Agent Framework

### Timing guidance
TODO

### Common pitfalls
TODO — **especially:** the hosted-agent (Modules 2–6) vs. Agent Framework (Module 7) distinction;
participants may expect the Module 2 agent to "become" the orchestrator rather than understanding
this is a separate, code-first agent definition and orchestration model. Also flag
`pip install agent-framework` version/compatibility issues.

### Talking points
TODO — include why Microsoft recommends Agent Framework over the now-retiring Prompt Flow.

## Module 8 — Evaluation

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., evaluator misconfiguration, dataset format errors)

### Talking points
TODO

## Module 9 — Monitoring & tracing

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., Application Insights connection string/instrumentation key confusion, missing spans)

### Talking points
TODO — note that Agent Framework emits OpenTelemetry spans natively, so Module 7's agents should
already be traced without extra instrumentation.

## Module 10 — Wrap-up & governance

### Timing guidance
TODO

### Common pitfalls
TODO (e.g., participants forgetting to delete resources, leading to ongoing cost)

### Talking points
TODO — cost awareness, responsible AI, cleanup steps, next-step learning paths.

## General workshop talking points

TODO: Add cross-cutting talking points (e.g., positioning Foundry vs. other Azure AI offerings,
how this scenario maps to real customer AP automation scenarios).

## Q&A / objection handling

TODO: Add anticipated participant questions and suggested answers.
