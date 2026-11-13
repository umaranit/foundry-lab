# Invoice Discrepancy Assistant — Microsoft Foundry Workshop Lab

## Scenario

Contoso Finance's accounts-payable team manually triages invoice discrepancies — price
mismatches, quantity mismatches, duplicate invoices, and missing purchase orders. In this
hands-on lab you'll build an **Invoice Discrepancy Assistant** on Microsoft Foundry that grounds
its answers in company policy, looks up purchase order (PO) records through a custom tool, calls
an MCP-based escalation-contact tool, and analyzes discrepancy trends — first as a Prompt
Agent configured in the Foundry portal, then as a code-first multi-agent system using the
Microsoft Agent Framework, then deploying a single specialist to Foundry Hosted Agents.

## Audience

Developers and solution architects with mixed prior experience with Microsoft Foundry.

## Total duration

~435 minutes (7 hours 15 minutes) across 13 modules (0–12). Each module lists an estimated time; actual pace
will vary by audience and Azure quota/approval wait times. This is best run as a full-day
workshop, or split across two sessions (e.g., Modules 0–6 on day one: ~195 minutes;
Modules 7–12 on day two: ~240 minutes, excluding breaks).

## How this lab is organized

The [modules/](/modules/) and [data/](/data/) folders are directly at the repository root.
Each module lives in [modules/](/modules/) as its own Markdown file, numbered in the order
you should complete them. Modules 5–7 extend the hosted agent created in Module 2: Module 5
builds the conversations/responses `client.py`, Module 6 adds async local MCP integration, and
Module 7 uses a separate policy-retrieval demo for Foundry IQ and remote MCP approvals, leaving
that client unchanged. Module 8 then introduces a separate, code-first Agent Framework system
running locally. Module 9 deploys a single specialist to Foundry's managed Hosted Agent container runtime, distinct from the Prompt Agent
in Modules 2–7. Modules 10–12 cover evaluation, monitoring/tracing, and wrap-up.
Sample data referenced throughout the lab is described in
[data/README.md](/data/README.md). Facilitators running this as an instructor-led workshop
should also review [FACILITATOR-NOTES.md](/FACILITATOR-NOTES.md).

## Modules

| # | Module | Objective | Est. time |
|---|--------|-----------|-----------|
| 0 | [Prerequisites](/modules/00-prerequisites.md) | Set up the Azure subscription, Foundry portal access, VS Code + Foundry Toolkit extension, Python, and Git needed for every later module. | 15 min |
| 1 | [Foundry project & model setup](/modules/01-foundry-project-and-model-setup.md) | Create a Foundry project and deploy a chat model, understanding the model selection and deployment options available in the portal. | 20 min |
| 2 | [Build the agent (portal)](/modules/02-build-the-agent-portal.md) | Create the Invoice Discrepancy Assistant as a Prompt Agent, write its system instructions, and test it in the playground. | 25 min |
| 3 | [Knowledge base / grounding — File Search](/modules/03-knowledge-base-grounding.md) | Upload the invoice/PO policy document and ground the agent's discrepancy-handling answers in it using File Search. | 30 min |
| 4 | [Tools — Built-in (Portal)](/modules/04-tools.md) | Add the Code Interpreter built-in tool to the agent via the Foundry portal so it can analyze discrepancy trends. (File Search was already added in Module 3.) | 15 min |
| 5 | [Custom Function Tools (Code)](/modules/05-programmatic-dev-vscode-hosted-agent.md) | Define a FunctionTool for "look up PO by number", create a new agent version with `project_client.agents.create_version`, and build a client using `openai_client.conversations.create()` / `openai_client.responses.create()` to call it. | 45 min |
| 6 | [MCP Integration (Code)](/modules/06-mcp-integration.md) | Build a local MCP server with FastMCP exposing an escalation-contact lookup tool grounded in the policy's Section 8 table, connect to it from the Module 5 client via the `mcp` package's `stdio_client`/`ClientSession` (wrapped as a `FunctionTool`, not an `MCPTool`/approval loop — that only applies to remote MCP servers), and convert the client to async. | 45 min |
| 7 | [Foundry IQ Knowledge Base (Code)](/modules/07-foundry-iq-knowledge-base.md) | Provision Azure AI Search and Blob Storage, connect Foundry IQ alongside File Search, require approval via the VS Code agent builder, and test approval/denial in a separate `foundry_iq_demo.py`. | 45 min |
| 8 | [Multi-agent orchestration with Microsoft Agent Framework](/modules/08-multi-agent-orchestration-agent-framework.md) | Define specialized code-first agents and an orchestrator using the Microsoft Agent Framework SDK, and contrast this code-first approach with the Prompt Agent from Modules 2–7. | 50 min |
| 9 | [Deploy to Foundry Hosted Agents](/modules/09-deploy-to-foundry-hosted-agents.md) | Package a single Module 8 specialist with the Responses hosting adapter, deploy with the Python SDK, wait for an active version, route and invoke it, then restore routing and delete the temporary version. | 45 min |
| 10 | [Evaluation](/modules/10-evaluation.md) | Capture real hosted-agent responses against a seed dataset and run the built-in `intent_resolution`, `relevance`, and `groundedness` cloud evaluators (no custom evaluator). | 45 min |
| 11 | [Monitoring & tracing](/modules/11-monitoring-and-tracing.md) | Wire up Application Insights / Azure Monitor, enable distributed tracing, and inspect traces/spans for token usage, latency, and tool calls. | 40 min |
| 12 | [Wrap-up & governance](/modules/12-wrapup-governance.md) | Review cost awareness and responsible AI considerations, clean up resources, and find next-step learning paths. | 15 min |

**Total:** ~435 minutes (7 hours 15 minutes), excluding breaks and additional provisioning buffers.

Module 7 supplies a separate Foundry IQ approval demo; Modules 5–6's `client.py` stays unchanged.
Module 9 supplies a single-specialist hosted entry point and deployment script, grounded in the
[official hosted-agent quickstart's Python SDK path](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent?pivots=python).
Full `HandoffBuilder` hosting remains explicitly unverified and is not part of Module 9.
Evaluation (Module 10) still targets the Module 6 Prompt Agent, and monitoring/tracing
(Module 11) retains its existing scope; neither depends on keeping Module 9's temporary version.

## Conventions used in this lab

- Steps are numbered and written for you to follow along in order.
- Code blocks contain placeholders such as `<your-resource-name>` — replace them with your own
  values.
- Each module ends with a **✅ Checkpoint / Validate** box so you can confirm you're on track
  before moving on, and a **🚀 Try it yourself** stretch challenge if you finish early.
- **⚠️ Troubleshooting** tips are called out where participants commonly get stuck.

## Scope notes

- This lab intentionally avoids **Prompt Flow** (retiring April 20, 2027) in favor of the
  **Microsoft Agent Framework** for code-first orchestration (Module 8).
- This lab intentionally avoids the Foundry portal's visual **Workflows** builder, which is
  still in preview; orchestration is done via the Prompt Agent (Modules 2–7) and via code
  (Module 8).

