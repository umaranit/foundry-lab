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
- Total duration: ~435 minutes (7 hours 15 minutes) across 13 modules (0–12), excluding breaks
  and additional provisioning buffers. Best split across two sessions
  (e.g., Modules 0–6 on day one: ~195 minutes; Modules 7–12 on day two: ~240 minutes,
  excluding breaks).
- Output: a set of Markdown files (one per module in modules/, plus an index/README at the
  repository root), with modules/ and data/ directly at the repository root, written as
  numbered step-by-step participant instructions (like the Microsoft Learn exercises),
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
2. Build the agent (portal) — create the Invoice Discrepancy Assistant as a Prompt Agent,
   write its system instructions, test in the playground.
3. Knowledge base / grounding — upload the invoice policy document, use File Search so the
   agent grounds discrepancy-handling answers in policy; explain indexing and citations.
4. Tools — Built-in (Portal) — add the Code Interpreter built-in tool via the Foundry portal so the
   agent can analyze the discrepancy trend CSV. (File Search was already added in Module 3.)
5. Custom Function Tools (Code) — define a FunctionTool for "look up PO by number" (reading
   po-records.csv), create a new agent version via project_client.agents.create_version with a
   PromptAgentDefinition that includes the tool, and build a client using
   openai_client.conversations.create() / openai_client.responses.create() (not threads/runs) to call
   it, handling function-call outputs.
6. MCP Integration (Code) — build a local MCP server using FastMCP with @mcp.tool() decorators
   exposing an escalation-contact lookup tool grounded in the policy's Section 8 table. Connect
   to the local subprocess with the mcp package's stdio_client / ClientSession, discover its
   tools, and wrap each discovered tool as a FunctionTool with parameters=t.inputSchema on a
   new Prompt Agent version. Extend Module 5's conversations/responses client with async
   local dispatch through session.call_tool and submit the function-call outputs. There is
   no MCPTool class and no approval loop here; remote MCP approvals belong to Module 7.
7. Foundry IQ Knowledge Base (Code) — 45 minutes. Ground the exercise in
   https://microsoftlearning.github.io/mslearn-ai-agents/Instructions/Exercises/04-integrate-agent-with-foundry-iq.html.
   Provision Azure AI Search and an Azure Storage account/container in the portal; upload
   invoice-po-policy.md to Blob Storage, optionally with a second fictional vendor onboarding
   policy. Use Knowledge > Create a knowledge base in Foundry, connect the container, and select
   embedding and chat completion models. Connect via the agent's Knowledge > Add > Connect to
   Foundry IQ, retaining Module 3's File Search. Configure the kb-knowledgebase tool to require
   approval in the Foundry Toolkit VS Code agent builder (Never auto-approve tools); the portal
   does not expose this setting and defaults to no approval. Supply a separate foundry_iq_demo.py
   using the conversations/responses pattern from Modules 5–6, leaving client.py unchanged.
   Retrieve the existing agent rather than creating a version; handle mcp_approval_request /
   mcp_approval_response until no requests remain, including denial and multiple requests.
   Contrast single-vector-store File Search with multi-source, agentic Foundry IQ retrieval and
   its Azure AI Search requirement. This is the remote MCP + approval pattern set aside in
   Module 6's local MCP integration. Compare actual tool use, citations, and policy correctness
   against the File Search-only baseline; the focused demo does not dispatch local function tools.
8. Multi-agent orchestration with Microsoft Agent Framework — define specialized code-first agents
   (price-mismatch-agent, duplicate-invoice-agent, missing-po-agent) using the Microsoft Agent Framework
   SDK (pip install agent-framework), and an orchestrator that classifies a discrepancy and routes it to
   the right specialist, then returns its result. Clearly call out how this differs from the Prompt
   Agent in Modules 2–7 (code-first definition and orchestration vs. server-hosted agent + portal config),
   and mention this is Microsoft's recommended direction over the now-retiring Prompt Flow.
9. Deploy to Foundry Hosted Agents — 45 minutes. Ground the exercise in
   https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent?pivots=python
   (Python SDK path). Explain Hosted Agents as managed container runtimes in Foundry Agent Service
   (AgentKind.HOSTED / HostedAgentDefinition), distinct from BOTH Modules 2–7's portal-configured
   Prompt Agent and Module 8's laptop-only Agent Framework script: this is deployment of custom
   agent code, not just prompt/tool configuration or a local script calling a cloud model.
   Adapt only Module 8's price-mismatch specialist into a separate main.py hosted entry point
   using ResponsesHostServer. Full HandoffBuilder hosting remains unverified and out of scope.
   Package source, dependencies, and required fictional PO data without secrets or laptop-only paths.
   Follow project_client.agents.create_version_from_code(...) with HostedAgentDefinition
   (cpu, memory, CodeConfiguration with runtime/entry_point/dependency_resolution,
   environment_variables, protocol_versions). Match wait_for_active_version by polling
   agents.get_version(...) for "active", handling "failed" and timeout explicitly.
   Save the previous endpoint config, route the new version via AgentEndpointConfig /
   VersionSelector / FixedRatioVersionSelectionRule, and invoke via
   project_client.get_openai_client(agent_name=...).responses.create(...).
   Mirror the quickstart's finally-based validate-then-clean-up lifecycle: restore the previous
   endpoint config when present and delete_version(..., force=True), confirming cleanup.
   Contrast this billable container version's immediate cleanup with Module 12's final
   resource-group deletion for model/project-level resources. Verify the quickstart's SDK/runtime,
   identity/permissions, and connected Application Insights prerequisites before deployment;
   do not defer these checks until Module 11. Supply the single-specialist entry point and
   source-deployment script. Keep Module 8's local script and Modules 10–11's existing targets unchanged.
10. Evaluation — capture real hosted-agent responses against a seed query/ground-truth dataset,
   then run the verified built-in Foundry cloud evaluators (intent_resolution, relevance,
   groundedness; no custom evaluator), interpret results.
11. Monitoring & tracing — wire up Application Insights / Azure Monitor, enable distributed tracing,
   inspect traces/spans for token usage, latency, and tool calls in the Foundry portal and locally. Note
   that Agent Framework emits OpenTelemetry spans natively, so Module 8's agents should already be traced.
12. Wrap-up & governance — cost awareness, responsible AI considerations, cleanup/resource deletion
    steps, and "where to go next" pointers to the full Microsoft Learn paths.

Also produce a short FACILITATOR-NOTES.md with timing guidance, common pitfalls per module (especially
the Prompt Agent vs. local Agent Framework vs. Hosted Agent distinction in Modules 5–9, and the conversations/responses vs.
threads/runs API distinction in Modules 5–7), and talking points for a live customer workshop.

Ask me clarifying questions before generating if anything about the scenario, Azure resources, or
module depth is ambiguous.