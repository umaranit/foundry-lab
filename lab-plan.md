You are an instructional designer and Microsoft Foundry solutions architect. Create a complete, hands-on workshop lab for customer-facing workshops, modeled on the Microsoft Learn exercise style (see
https://microsoftlearning.github.io/mslearn-ai-agents/Instructions/Exercises/01-build-agent-portal-and-vscode.html,
https://microsoftlearning.github.io/mslearn-ai-agents/Instructions/Exercises/08-agent-framework-multi-agents.html,
and https://microsoftlearning.github.io/mslearn-genaiops/ for tone, structure, and depth).

SCENARIO
Build an "Invoice Discrepancy Assistant" solution for a fictional company (e.g., Contoso Finance) that
triages accounts-payable invoice discrepancies: price mismatches, quantity mismatches, duplicate
invoices, and missing purchase orders. The agent should ground its answers in an invoice/PO policy
document, look up PO records via a custom tool, and analyze discrepancy trends from a sample dataset.

AUDIENCE & FORMAT
- Audience: developers and solution architects, mixed Foundry experience.
- Total duration: ~345 minutes (~5.75 hours), split into clearly labeled modules with time
  estimates. Best run as a full-day workshop, or split across two half-day sessions (e.g.,
  Modules 0–5 on day one, Modules 6–10 on day two).
- Output: a set of Markdown files (one per module, plus an index/README) in a folder called
  /lab, written as numbered step-by-step participant instructions (like the Microsoft Learn exercises),
  each with: objective, prerequisites, numbered steps, code blocks with placeholders, a "checkpoint/validate"
  box, a "try it yourself" stretch challenge, and common troubleshooting tips.
- Also generate the sample data files referenced in the steps (invoice policy doc, sample PO records CSV,
  sample invoice discrepancy test dataset for evaluation) with realistic placeholder content.
- Avoid features that are in preview or being retired unless explicitly called out as optional/experimental
  (e.g., do NOT use Prompt Flow — it is retiring April 20, 2027 in favor of Microsoft Agent Framework; do
  NOT use the Foundry portal's visual Workflows builder — it is still in preview).

MODULES TO COVER (one markdown file each, in this order)
0. Prerequisites — Azure subscription, Foundry portal access, VS Code + Foundry Toolkit extension,
   Python 3.13, Git, quota considerations.
1. Foundry project & model setup — create a Foundry project, deploy a chat model, explain model
   selection/deployment options in the portal.
2. Build the agent (portal) — create the Invoice Discrepancy Assistant as a hosted Prompt Agent,
   write its system instructions, test in the playground.
3. Knowledge base / grounding — upload the invoice policy document, use File Search / Foundry IQ so the
   agent grounds discrepancy-handling answers in policy; explain indexing and citations.
4. Tools — Built-in (Portal) — add the Code Interpreter built-in tool via the Foundry portal so the
   agent can analyze the discrepancy trend CSV. (File Search was already added in Module 3.)
5. Custom Function Tools (Code) — define a FunctionTool for "look up PO by number" (reading
   po-records.csv), create a new agent version via project_client.agents.create_version with a
   PromptAgentDefinition that includes the tool, and build a client using
   openai_client.conversations.create() / openai_client.responses.create() (not threads/runs) to call
   it, handling function-call outputs.
6. MCP Integration (Code) — build a local MCP server using FastMCP with @mcp.tool() decorators
   exposing an escalation-contact lookup tool grounded in the policy's Section 8 table. Add it to the
   agent as an MCPTool (new agent version), and extend the Module 5 client to handle MCP approval
   requests in the conversations/responses loop.
7. Multi-agent orchestration with Microsoft Agent Framework — define specialized code-first agents
   (price-mismatch-agent, duplicate-invoice-agent, missing-po-agent) using the Microsoft Agent Framework
   SDK (pip install agent-framework), and an orchestrator that classifies a discrepancy and routes it to
   the right specialist, then combines results. Clearly call out how this differs from the hosted Prompt
   Agent in Modules 2–6 (code-first definition and orchestration vs. server-hosted agent + portal config),
   and mention this is Microsoft's recommended direction over the now-retiring Prompt Flow.
8. Evaluation — capture real hosted-agent responses against a seed query/ground-truth dataset,
   then run the verified built-in Foundry cloud evaluators (intent_resolution, relevance,
   groundedness; no custom evaluator), interpret results.
9. Monitoring & tracing — wire up Application Insights / Azure Monitor, enable distributed tracing,
   inspect traces/spans for token usage, latency, and tool calls in the Foundry portal and locally. Note
   that Agent Framework emits OpenTelemetry spans natively, so Module 7's agents should already be traced.
10. Wrap-up & governance — cost awareness, responsible AI considerations, cleanup/resource deletion
    steps, and "where to go next" pointers to the full Microsoft Learn paths.

Also produce a short FACILITATOR-NOTES.md with timing guidance, common pitfalls per module (especially
the hosted-agent vs. Agent Framework distinction in Modules 5–7, and the conversations/responses vs.
threads/runs API distinction in Modules 5–6), and talking points for a live customer workshop.

Ask me clarifying questions before generating if anything about the scenario, Azure resources, or
module depth is ambiguous.