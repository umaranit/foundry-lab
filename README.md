# Invoice Discrepancy Assistant — Microsoft Foundry Workshop Lab

## Scenario

Contoso Finance's accounts-payable team manually triages invoice discrepancies — price
mismatches, quantity mismatches, duplicate invoices, and missing purchase orders. In this
hands-on lab you'll build an **Invoice Discrepancy Assistant** on Microsoft Foundry that grounds
its answers in company policy, looks up purchase order (PO) records through a custom tool, calls
an MCP-based escalation-contact tool, and analyzes discrepancy trends — first as a hosted Prompt
Agent configured in the Foundry portal, then as a code-first multi-agent system using the
Microsoft Agent Framework.

## Audience

Developers and solution architects with mixed prior experience with Microsoft Foundry.

## Total duration

~345 minutes (~5.75 hours) across 11 modules. Each module lists an estimated time; actual pace
will vary by audience and Azure quota/approval wait times. This is best run as a full-day
workshop, or split across two half-day sessions (e.g., Modules 0–5 on day one, Modules 6–10 on
day two).

## How this lab is organized

Each module lives in [modules/](/lab/modules) as its own Markdown file, numbered in the order
you should complete them (Modules 5–7 in particular depend on the hosted agent created in
Module 2, so do not skip ahead). Sample data referenced throughout the lab is described in
[data/README.md](/lab/data/README.md). Facilitators running this as an instructor-led workshop
should also review [FACILITATOR-NOTES.md](/lab/FACILITATOR-NOTES.md).

## Modules

| # | Module | Objective | Est. time |
|---|--------|-----------|-----------|
| 0 | [Prerequisites](/lab/modules/00-prerequisites.md) | Set up the Azure subscription, Foundry portal access, VS Code + Foundry Toolkit extension, Python, and Git needed for every later module. | 15 min |
| 1 | [Foundry project & model setup](/lab/modules/01-foundry-project-and-model-setup.md) | Create a Foundry project and deploy a chat model, understanding the model selection and deployment options available in the portal. | 20 min |
| 2 | [Build the agent (portal)](/lab/modules/02-build-the-agent-portal.md) | Create the Invoice Discrepancy Assistant as a hosted Prompt Agent, write its system instructions, and test it in the playground. | 25 min |
| 3 | [Knowledge base / grounding](/lab/modules/03-knowledge-base-grounding.md) | Upload the invoice/PO policy document and ground the agent's discrepancy-handling answers in it using File Search / Foundry IQ. | 30 min |
| 4 | [Tools — Built-in (Portal)](/lab/modules/04-tools.md) | Add the Code Interpreter built-in tool to the agent via the Foundry portal so it can analyze discrepancy trends. (File Search was already added in Module 3.) | 15 min |
| 5 | [Custom Function Tools (Code)](/lab/modules/05-programmatic-dev-vscode-hosted-agent.md) | Define a FunctionTool for "look up PO by number", create a new agent version with `project_client.agents.create_version`, and build a client using `openai_client.conversations.create()` / `openai_client.responses.create()` to call it. | 45 min |
| 6 | [MCP Integration (Code)](/lab/modules/06-mcp-integration.md) | Build a local MCP server with FastMCP exposing an escalation-contact lookup tool grounded in the policy's Section 8 table, connect to it from the Module 5 client via the `mcp` package's `stdio_client`/`ClientSession` (wrapped as a `FunctionTool`, not an `MCPTool`/approval loop — that only applies to remote MCP servers), and convert the client to async. | 45 min |
| 7 | [Multi-agent orchestration with Microsoft Agent Framework](/lab/modules/07-multi-agent-orchestration-agent-framework.md) | Define specialized code-first agents and an orchestrator using the Microsoft Agent Framework SDK, and contrast this code-first approach with the hosted Prompt Agent from Modules 2–6. | 50 min |
| 8 | [Evaluation](/lab/modules/08-evaluation.md) | Capture real hosted-agent responses against a seed dataset and run the built-in `intent_resolution`, `relevance`, and `groundedness` cloud evaluators (no custom evaluator). | 45 min |
| 9 | [Monitoring & tracing](/lab/modules/09-monitoring-and-tracing.md) | Wire up Application Insights / Azure Monitor, enable distributed tracing, and inspect traces/spans for token usage, latency, and tool calls. | 40 min |
| 10 | [Wrap-up & governance](/lab/modules/10-wrapup-governance.md) | Review cost awareness and responsible AI considerations, clean up resources, and find next-step learning paths. | 15 min |

**Total:** ~345 minutes (~5.75 hours), excluding breaks.

## Conventions used in this lab

- Steps are numbered and written for you to follow along in order.
- Code blocks contain placeholders such as `<your-resource-name>` — replace them with your own
  values.
- Each module ends with a **✅ Checkpoint / Validate** box so you can confirm you're on track
  before moving on, and a **🚀 Try it yourself** stretch challenge if you finish early.
- **⚠️ Troubleshooting** tips are called out where participants commonly get stuck.

## Scope notes

- This lab intentionally avoids **Prompt Flow** (retiring April 20, 2027) in favor of the
  **Microsoft Agent Framework** for code-first orchestration (Module 7).
- This lab intentionally avoids the Foundry portal's visual **Workflows** builder, which is
  still in preview; orchestration is done via the hosted Prompt Agent (Modules 2–6) and via code
  (Module 7).

> **Open question:** see [FACILITATOR-NOTES.md](/lab/FACILITATOR-NOTES.md) and the "Open
> questions / ambiguities" note at the bottom of this scaffolding pass for items that need
> confirmation before full module content is written.
