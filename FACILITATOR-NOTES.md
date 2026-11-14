# Facilitator Notes — Invoice Discrepancy Assistant Lab

Use these notes alongside the participant modules. The timings are planning estimates, not Azure
provisioning guarantees. Keep the modules' explicit SDK/runtime verification caveats visible;
in particular, Module 8's workflow-result handling and full multi-agent hosting are not implied
to be verified by completion of the workshop documentation.

## How to use this document

- **Before the workshop:** Rehearse the [Module 0](modules/00-prerequisites.md) access and quota
  checks in the intended subscription/region. Arrange approvals before the event; a pending
  quota request is not usable capacity. Read policy Sections 3–8 and the sample PO records so
  you can check decisions, not just fluent prose. Use only the supplied fictional data.
- **Preflight the slow paths:** Confirm Search/Storage permissions and supported embedding/chat
  deployments for [Module 7](modules/07-foundry-iq-knowledge-base.md). Rehearse
  [Module 9](modules/09-deploy-to-foundry-hosted-agents.md)'s isolated environment, runtime,
  permissions, connected Application Insights, and source build. Delete rehearsal Hosted Agent
  versions immediately. If preparing shared resources, record ownership and who can change
  agent configuration or delete resources; each hosted deployment needs its own dedicated name.
- **During the workshop:** Use each module's checkpoint as the exit criterion. Ask participants
  to identify where the agent and tool code execute at the transitions into Modules 5, 8, and 9.
  Keep Modules 5–6's client, Module 7's separate approval demo, Module 8's local workflow, and
  Module 9's isolated hosted source distinct. Before Modules 10–11, confirm the intended Module 6
  Prompt Agent/tool configuration; their scripts are not tests of the Module 9 deployment or
  dispatchers for Module 7's remote approvals.
- **After the exercises:** Reserve Module 12 for actual cleanup verification, not just a recap.
  Check Module 9's temporary version is already gone, then account for Search, Storage, model
  deployments, telemetry, and any resources outside the workshop resource group. Keep only
  authorized, non-secret evaluation/trace observations needed for follow-up.

## Overall timing guidance

The module estimates total **435 minutes (7 hours 15 minutes)** of exercises, before additional
buffers and breaks. Modules 7 and 9 are full exercises, not outline discussion slots. Reserve the
extra time below for a mixed-experience group; some waits can overlap discussion, but quota
approval or unsupported regional capacity cannot be solved by adding a few minutes.

**Recommended run-of-show:** split into two sessions: Modules 0–6 total **195 minutes**, and
Modules 7–12 total **240 minutes**, before buffers. Place a 15-minute break after Module 4,
another after Module 8, and another after Module 10 (while the evaluation runs, if appropriate).
All listed buffers add 95–155 minutes; with those three breaks the complete workshop needs
roughly **9 hours 35 minutes to 10 hours 35 minutes**. An ordinary single-day slot needs
prerequisite work and resource pre-provisioning completed in advance or a smaller hands-on scope.

If behind, omit stretch challenges, the second-model comparison in Module 1, extra charts in
Module 4, and extra prompt variations. Keep at least one observable tool call in Modules 5–6,
approval **and denial** in Module 7, all three routing cases in Module 8, and readiness,
invocation, and cleanup in Module 9. Use a labeled facilitator demonstration if provisioning
blocks a participant; do not mark their deployment checkpoint complete. Never trim cleanup.

| Module | Planned time | Buffer notes |
|---|---|---|
| 0 — Prerequisites | 15 min | +5 min for sign-in/interpreter checks; permissions and quota approvals must precede the event |
| 1 — Foundry project & model setup | 20 min | +5–10 min for project/model provisioning and regional capacity issues |
| 2 — Build the agent (portal) | 25 min | +5 min for agent save/playground response delays |
| 3 — Knowledge base / grounding | 30 min | +5–10 min for policy upload, indexing, and citation checks |
| 4 — Tools — Built-in (Portal) | 15 min | +5 min for CSV upload and first Code Interpreter session |
| 5 — Custom Function Tools (Code) | 45 min | +5–10 min for packages, credentials, paths, and tool-loop debugging |
| 6 — MCP Integration (Code) | 45 min | +5–10 min for async conversion and local subprocess/venv issues |
| 7 — Foundry IQ Knowledge Base (Code) | 45 min | +15–25 min for Search/Storage provisioning, model setup, and knowledge-source indexing |
| 8 — Multi-agent orchestration (Agent Framework) | 50 min | +10 min for package/result-shape checks and routing diagnosis |
| 9 — Deploy to Foundry Hosted Agents | 45 min | +15–25 min for remote dependency build, container activation, identity propagation, and cleanup recovery |
| 10 — Evaluation | 45 min | +10–20 min for capture calls, judge quota, and cloud evaluation queues |
| 11 — Monitoring & tracing | 40 min | +5–10 min for telemetry connection/export setup and ingestion delay |
| 12 — Wrap-up & governance | 15 min | +5–10 min for deletion confirmation and shared-resource ownership checks |

## Module 0 — Prerequisites

### Timing guidance
Use 5 minutes for subscription/portal sign-in, 5 for quota and region checks, and 5 for Python,
VS Code/Foundry Toolkit, Git, and the working folder. Allow 5 extra minutes for local setup.
Move installations and access requests before the workshop; quota approval can outlast the event.

### Common pitfalls
- Subscription access or quota approvals were not requested early enough. Have an authorized,
  pre-provisioned project option rather than promising approval during Module 1.
- Browser and CLI target different tenants/subscriptions, or `python` resolves to an older
  interpreter. Confirm the selected account and Python 3.13 before creating environments.

### Talking points
- A portal sign-in, permission to create resources, and available model quota are separate checks.
- Region affects model availability and later Hosted Agent/Search capabilities; select it for the
  full lab, not just the first playground call.
- The policy, PO records, trend CSV, and evaluation seed serve different purposes; none requires
  real customer financial data.

## Module 1 — Foundry project & model setup

### Timing guidance
Use 5 minutes to create/tour the project, 5 to compare model options, and 10 to deploy and test
the selected model. Reserve 5–10 additional minutes for Azure provisioning. Use that wait to
compare deployment cost/residency options; unresolved quota needs a supported fallback.

### Common pitfalls
- Model/region availability and quota can block deployment even with resource-creation access.
- Participants confuse a model catalog name with their deployment name, or copy the wrong
  endpoint for later SDK calls. Record the actual project endpoint and deployment name.

### Talking points
- Select for tool-use capability, cost, context window, and available capacity, not just model size.
- Standard pay-as-you-go and provisioned throughput solve different utilization needs; the lab
  does not need reserved throughput.
- Global/regional deployment choices and model versioning affect residency, latency, and evaluation
  reproducibility. Use the New Foundry experience throughout.

## Module 2 — Build the agent (portal)

### Timing guidance
Use 5 minutes for agent creation, 10 for instructions and the execution-model distinction, and
10 for the three baseline prompts and saving. Allow 5 extra minutes for playground delays.
Keep an ungrounded baseline answer for the Module 3 comparison.

### Common pitfalls
- The agent cannot retrieve policy thresholds or PO records yet. A plausible answer is not proof
  of access; the expected behavior is to acknowledge missing information.
- A model deployment from another project will not appear here. Confirm the same project and
  successful deployment before changing instructions.

### Talking points
- This is a **Prompt Agent**, `AgentKind.PROMPT`: Foundry manages its execution. Module 9's
  **Hosted Agent**, `AgentKind.HOSTED`, instead runs packaged custom code on managed compute.
- Instructions establish role, scope, and refusal to invent facts; they do not supply a data
  connection or enforce a financial approval gate.
- If using the hard-coded-threshold stretch exercise, revert it before Module 3. Policy values
  belong in the authoritative source, not a second, drifting copy of the rules.

## Module 3 — Knowledge base / grounding

### Timing guidance
Use 5 minutes to inspect the policy, 10 to upload/attach File Search and update instructions,
and 15 to test policy answers and citations. Allow 5–10 extra minutes for indexing or retries;
upload completion alone is not retrieval readiness.

### Common pitfalls
- Indexing delays or an empty/wrong vector store can look like poor prompting.
- Citations may be collapsed in the playground. Expand the source and verify the actual policy
  section rather than accepting a document name in prose.

### Talking points
- Check both price conditions: **≤ 2% and ≤ $25**. A correct-looking threshold with the wrong
  logical operator is still a wrong answer.
- File Search retrieves evidence; citations let the analyst inspect it. Neither guarantees the
  model applies every rule correctly.
- This exercise uses File Search. Despite the earlier module's grouped terminology, Module 7
  explicitly distinguishes the separately provisioned Azure AI Search-backed Foundry IQ path.

## Module 4 — Tools — Built-in (Portal)

### Timing guidance
Use 5 minutes to attach Code Interpreter and upload the trend CSV, then 10 for aggregation,
vendor follow-up, and a policy-plus-analysis question. Reserve 5 extra minutes for file readiness
and first execution. The chart challenge can be omitted without losing the core demonstration.

### Common pitfalls
- Code Interpreter file uploads are separate from File Search attachments; putting the CSV in
  the knowledge source does not make it available to the execution session.
- Custom function and MCP tools come in Modules 5 and 6, not this portal-only exercise.

### Talking points
- File Search retrieves policy text; Code Interpreter runs generated code for aggregation and
  charts. Inspect the execution step, not just the asserted trend.
- The trend dataset is not the individual-PO lookup table. Each tool needs the appropriate data.
- A combined question can need both policy evidence and calculated counts; this is still the
  same Prompt Agent, not a multi-agent workflow.

## Module 5 — Custom Function Tools (Code)

### Timing guidance
Use 10 minutes for the environment and data path, 10 for the lookup/schema, 10 for the new agent
version and retained built-ins, and 15 for the conversations/responses loop and tests. Allow
5–10 extra minutes for package/authentication or schema issues; no new Azure service is required.

### Common pitfalls
- Use `openai_client.conversations.create()` / `openai_client.responses.create()`, not legacy
  threads/runs examples.
- Changing a local tool definition does not update the service until
  `project_client.agents.create_version` saves a new version. Retain File Search and Code
  Interpreter; schema names/arguments must match the local dispatcher.
- `LAB_REPO_ROOT` must identify the folder containing the sample data, and the terminal process
  must have the needed environment values. PowerShell does not load `.env` automatically.

### Talking points
- The Prompt Agent remains in Foundry. `FunctionTool` describes the call; the local client
  executes the CSV lookup and sends the result back. Registering a schema does not host Python.
- A nonexistent PO is a valid business outcome, not a guessed record or an infrastructure error.
  Test both PO-10234 and a missing record such as PO-10243.
- This is a new version of the same agent, not a new framework/runtime; Module 8 changes that.

## Module 6 — MCP Integration (Code)

### Timing guidance
Use 10 minutes to inspect Section 8 and build/start FastMCP, 15 to convert the client to async
and discover/wrap tools, 10 for version creation and dispatch, and 10 for lookup tests.
Reserve 5–10 extra minutes for subprocess/venv or async mistakes, not Azure provisioning.

### Common pitfalls
- Keep `async with` / awaited MCP calls inside `main()` and call `asyncio.run(main())`. The
  existing synchronous Azure/OpenAI calls are not awaited.
- Launch from the working folder and activated venv containing `fastmcp`/`mcp`; the subprocess
  uses the terminal's `python`, not automatically VS Code's selected interpreter.
- Escalation lookup keys and the parser must match the policy's Section 8 table structure.
  A directly launched stdio server waiting silently for a client is expected, not a hang.

### Talking points
- FastMCP exposes the tool; the `mcp` package's `stdio_client` and `ClientSession` discover and
  call it. Each discovered tool becomes a `FunctionTool` with `parameters=t.inputSchema`.
- This is another Prompt Agent version using Module 5's conversations/responses pattern.
  The local client dispatches to the subprocess and returns function outputs; there is no
  `MCPTool` class or approval loop in this integration.
- Reading Section 8 at call time avoids a second contact list drifting from policy. Module 7
  contrasts this local adapter with remote MCP retrieval and explicitly enabled approval.

## Module 7 — Foundry IQ Knowledge Base (Code)

### Timing guidance
Use 5 minutes to capture the File Search baseline, 15 for Search/Storage provisioning and upload,
10 for knowledge-base connections/indexing, 10 for the separate approval demo, and 5 to compare
results. Add **15–25 minutes** for Search and Storage provisioning, embedding/chat deployment
readiness, and knowledge-source indexing. These are known slow steps; start them promptly and
discuss retrieval trade-offs while they run. Wait for **active** rather than treating save as ready.

### Common pitfalls
- Search/Storage provisioning and indexing can consume the buffer. Confirm supported region,
  tier, authentication, and model deployments; do not silently select a more expensive tier.
- **Pre-workshop policy check:** Verify that key authentication is permitted for Azure AI Search
  and Storage in the target subscription/tenant, via a quick resource-creation test or the
  subscription's policy owner. Azure Policy can disable key-based authentication entirely,
  blocking Step 2's **API Access control = Both** and the Search connection's **Key authentication**
  regardless of participant actions. If blocked, skip Module 7 for that cohort or arrange an
  approved sandbox subscription without the restrictive policy.
- Configure **Never auto-approve tools** on **`kb-knowledgebase...`** in the VS Code agent builder
  and **Deploy** it. The portal does not expose that setting; Web search and standalone Azure
  AI Search are the wrong tools.
- Retain both knowledge tools. Rerunning the older client's version-creation code can omit the
  knowledge base and approval configuration; use the separate `foundry_iq_demo.py`.
- The approval loop must cover every request and repeated rounds. Zero requests can mean
  File Search answered; denial can still be followed by a File Search answer. Inspect actual use.

### Talking points
- Contrast one File Search vector store with multi-source-capable, agentic Foundry IQ retrieval.
  This lab still uses the same policy document, so equal answers are a valid result, not a failure.
- Approval is explicitly enabled here, not the portal default. Foundry executes remote
  retrieval after the client's decision; this is not Module 6's local subprocess dispatch.
- The separate demo retrieves the existing agent rather than replacing it, and deliberately does
  not execute its local function tools. Use policy-only prompts and test both approval and denial.
- Compare citations, correctness, observed tool use, and latency against added Search, Storage,
  embedding, and retrieval-model cost. More infrastructure is not automatic evidence of better answers.

## Module 8 — Multi-agent orchestration with Microsoft Agent Framework

### Timing guidance
Use 5 minutes for package/authentication checks, 10 for `@tool` and the chat client, 15 for
specialists/coordinator, 10 for `HandoffBuilder`, and 10 for the three routing cases. Reserve
10 extra minutes for SDK compatibility and the module's explicit workflow-result-shape check.
No application is deployed to Azure in this module.

### Common pitfalls
- The Module 2 Prompt Agent does not become the orchestrator. These are separate code-defined
  agents with no shared agent ID or automatic inheritance of its knowledge/tools.
- Check the current `agent-framework` installation requirements and installed result API;
  Module 8 explicitly flags final-output extraction as needing verification.
- Specialist names must match the coordinator's targets. The module uses `AzureCliCredential`,
  so a valid CLI sign-in is required even if another credential source worked earlier.

### Talking points
- `@tool`, `Annotated`, and `Field` derive the function schema; orchestration and tool execution
  run locally while model inference remains in Foundry.
- `HandoffBuilder` selects one specialist for each supplied case; `SequentialBuilder` would run
  a fixed pipeline. This demo is not an ensemble and has no quantity-mismatch specialist.
- Policy is embedded in specialist instructions here, not inherited File Search grounding.
  Compare routing and factual decisions, not just whether the script produces output.
- Agent Framework is the lab's recommended code-first orchestration path instead of Prompt Flow,
  which retires April 20, 2027. That choice does not make every evolving framework API verified.

## Module 9 — Deploy to Foundry Hosted Agents

### Timing guidance
Use 5 minutes for prerequisites, 10 for isolated source/dependencies/data packaging, 5 for the
local HTTP test, 15 for remote build, activation, routing, and invocation, and 10 for cleanup
verification. Add **15–25 minutes** for remote dependency resolution/build, container startup,
identity propagation, or cleanup recovery. The script's roughly 10-minute activation polling
limit is a failure boundary, not a guarantee of build duration; investigate a timeout and confirm
cleanup before retrying. Do not let a slow build consume the reserved cleanup time.

### Common pitfalls
- `HostedAgentDefinition` configures container hosting, not a Prompt Agent. Renaming Module 8's
  batch script to `main.py` does not provide an HTTP service; use `ResponsesHostServer`.
- Deploy only the documented single price-mismatch specialist. Full `HandoffBuilder` hosting
  remains unverified and is outside this exercise.
- Check SDK/runtime compatibility, packaged PO data paths, the hosted identity's model access,
  and connected Application Insights **before** deployment, not in Module 11. Never package
  laptop credentials or `.env`.
- Use a dedicated agent name: routing affects the whole endpoint. Wait for `"active"` and
  handle `"failed"`/timeout; an allocated version number is not readiness.
- Endpoint restoration or deletion failures are failed cleanup, not warnings to ignore. Keep the
  exact name/version and saved routing for recovery; killing the process can bypass `finally`.

### Talking points
- The code now runs on managed container compute, unlike Prompt Agent configuration or a
  laptop calling a cloud model. The CSV lookup moves with it; the local MCP server does not.
- The source ZIP is built remotely. Readiness, correct routing, and a correct business answer
  are separate checks: the main test has about 4.05% variance but **$600** total impact, so
  Procurement escalation applies despite the percentage being below 5%.
- Follow the [Python SDK quickstart](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent?pivots=python)
  lifecycle: create, wait, save/route, invoke, restore prior routing when present, and delete the
  temporary version. A remaining dedicated agent record is not a running deployment.
- Hosted compute is separately billable. Verify deletion **now**, not at Module 12. Modules
  10–11 retain their Prompt Agent/local-workflow targets and do not need this version running.

## Module 10 — Evaluation

### Timing guidance
Use 5 minutes to inspect the 15-row seed and evaluation target, 15 to capture real responses,
15 to upload/run the three evaluators, and 10 to inspect low-scoring rows. Reserve 10–20 extra
minutes for model quota and cloud queues. Start evaluation before a break; if it is still running,
record its run ID and return to it after tracing rather than repeatedly submitting billable runs.

### Common pitfalls
- The seed has no responses. Capture real answers first, with a fresh conversation per row,
  and recapture after an agent change; otherwise evaluation scores stale answers.
- Match evaluator names, item schema, and field mappings exactly. Inspect errored items and the
  judge deployment rather than interpreting a partially failed run as a quality result.
- Bump the dataset version when uploading new captured answers; reusing a registered version
  can reuse the previous data.
- Confirm the Module 6 tool-dispatch target. These scripts do not implement Module 7's remote
  approval loop and do not evaluate the Module 8 orchestrator or Module 9 specialist.

### Talking points
- `intent_resolution`, `relevance`, and `groundedness` answer different questions; examine
  individual responses and evaluator reasoning, not only an average.
- Here `groundedness` receives `ground_truth` as context. It measures answer-vs-reference
  similarity, **not strict retrieval-groundedness** against the actual File Search evidence.
  Production evaluation should capture retrieved context/citations per query.
- Fifteen rows demonstrate the pipeline, not statistical confidence or production acceptance.
  The same deployment acting as both agent model and judge is another lab simplification.
- Capture and judging both consume tokens; better evaluation coverage has a real cost.

## Module 11 — Monitoring & tracing

### Timing guidance
Use 10 minutes for packages and the Application Insights connection/exporter, 10 to instrument
and run three queries, 10 to inspect resource metrics and a parent/child trace, and 10 for the
Agent Framework comparison. Add 5–10 minutes for connection setup or ingestion. Discuss query
duration versus per-call latency while waiting; an empty immediate view is not proof of failure.

### Common pitfalls
- Use the full Application Insights connection string, not an instrumentation key alone, and
  confirm the project connection rather than assuming it exists.
- Missing spans can mean ingestion delay, the wrong project/time filter, or exporter setup
  after model calls. Agent Framework emitting spans does not configure export in every process.
- The raw-client demo's token attributes depend on response usage availability, as the module
  notes. Missing token attributes are not zero cost or proof the entire trace failed.

### Talking points
- The raw Responses client uses `OpenAIInstrumentor` plus one manual parent span per query.
  Multiple model-call children expose tool round trips and their latency.
- Agent Framework emits OpenTelemetry spans natively, so it does not need comparable manual
  spans or that instrumentor; it still needs a configured exporter in its own process.
- Evaluation measures answer quality; traces help explain execution, duration, and token use.
  Module 11 makes separate sample runs, not automatic trace correlation to Module 10's rows.
- Telemetry can contain queries, PO numbers, and amounts. Access and retention belong in the
  same governance conversation as the financial source data.

## Module 12 — Wrap-up & governance

### Timing guidance
Use 5 minutes for execution-model/cost recap, 5 for approval authority and data governance, and
5 to initiate and check cleanup. Reserve 5–10 additional minutes for asynchronous deletion and
shared-resource ownership checks. If deletion is still pending, name the responsible owner and
follow up until absence is confirmed; `--no-wait` is not a deletion confirmation.

### Common pitfalls
- Closing a terminal does not delete Search, Storage, or Hosted Agent compute. Confirm Module
  9's temporary version was removed even if its invocation failed.
- Deleting the Prompt Agent alone does not remove all billable resources. Inventory Module 7's
  additions and resources outside the lab resource group; never delete a shared group without
  its owner's approval.

### Talking points
- Use Module 12's model-token, Code Interpreter, storage, and telemetry cost discussion, **plus**
  Module 7's Search/Storage/embedding/retrieval charges and Module 9's separately billed compute.
  Do not describe every non-model resource as a fixed negligible cost: provisioned Search and
  a running Hosted Agent version can cost money after the client exits.
- The assistant recommends; it does not release funds. Section 7's approval-authority ceiling
  still applies even when a discrepancy is within another section's tolerance.
- Move from fictional CSVs to an authorized ERP lookup only with access controls, auditability,
  representative evaluation, and human-owned payment gates. Use the linked Learn paths for
  deeper agent development and GenAIOps, not the lab's small sample as production certification.

## General workshop talking points

### Four execution models, not four Foundry agent kinds

| Lab pattern | Where execution happens | What the participant changes |
|---|---|---|
| Modules 2–4: portal-configured Prompt Agent | Foundry manages agent execution and built-in tools | Instructions, model selection, File Search, Code Interpreter |
| Modules 5–7: Prompt Agent with code-driven tool integration | Foundry runs the Prompt Agent; the client executes local functions/MCP in 5–6; Foundry executes remote IQ retrieval after the configured approval in 7 | Agent versions and local dispatch, then a separate remote-approval demo |
| Module 8: local Agent Framework system | Laptop runs coordinator, specialists, and tools; Foundry supplies model inference | Python agent definitions, `@tool`, and `HandoffBuilder` |
| Module 9: Hosted Agent | Foundry managed container runs the single specialist and packaged CSV tool | Source, dependencies, hosting adapter, runtime/compute, routing, lifecycle |

- **Terminology matters:** the first two patterns are `AgentKind.PROMPT`, not
  `AgentKind.HOSTED`. Agent Framework is a programming/orchestration choice, not a third
  Foundry agent kind; Module 9 demonstrates hosting one of its specialists.
- **Foundry does not replace the business system:** the project connects models, agent
  definitions, knowledge, evaluation, and telemetry. Search and Storage serve retrieval; an ERP
  would remain authoritative for real PO records and payment state.
- **Trust the evidence appropriate to each claim:** a citation supports a policy lookup, a
  function result supports a PO fact, a trace supports a tool-use claim, and evaluator scores
  summarize behavior on the supplied data. None alone certifies end-to-end financial correctness.
- **Approval has multiple meanings:** Module 7 approves a retrieval call; Module 8's autonomous
  handoff is an orchestration setting; policy Sections 2/7 govern financial authorization.
  Neither tool approval nor an autonomous handoff authorizes a payment.
- **Complexity must earn its cost:** a second retrieval path or coordinator adds calls, latency,
  and operational dependencies. Compare observed benefits before choosing multi-agent or
  Foundry IQ architecture for a small policy pack.
- **Production gaps remain explicit:** the lab uses fictional data, local CSV tools, a small
  evaluation set, and some embedded policy instructions. Production needs governed data access,
  actual retrieval-context evaluation, failure handling, and cost/telemetry controls.

## Q&A / objection handling

- **"Isn't every cloud agent a Hosted Agent?"** No. Prompt Agent means Foundry executes the
  prompt/tool definition. Hosted Agent is the distinct custom-code container kind deployed in
  Module 9; use the kind names rather than "hosted" as a casual prefix.
- **"Did registering a FunctionTool upload my function?"** No. In Modules 5–6, the schema tells
  the model what it can request; the client must execute the function or call the local MCP
  server and submit the result. Module 9 explicitly packages code for remote execution.
- **"Why no MCP approval in Module 6?"** Its local stdio server is adapted to ordinary
  FunctionTools and dispatched by the client. Module 7 uses a remote knowledge tool and
  explicitly enables approval. These examples do not establish a universal MCP security policy.
- **"I denied Foundry IQ, but got an answer. Did denial fail?"** Not necessarily. File Search
  remains available and prior conversation content may help. Check tool evidence in a fresh
  conversation; the agent must not claim it consulted the denied source.
- **"Should I replace File Search with Foundry IQ?"** Not automatically. One small policy may
  work well with File Search. Module 7 retains both to compare; multi-source retrieval must
  justify Search, Storage, embedding, retrieval-model cost, and operational complexity.
- **"Why Agent Framework instead of Prompt Flow or a sequential chain?"** Prompt Flow is retiring;
  Agent Framework is the lab's code-first direction. A handoff matches the three single-specialist
  test cases; a sequential chain would execute unnecessary participants. Real mixed-discrepancy
  invoices require additional routing design and evaluation.
- **"Did Module 9 deploy the whole workflow?"** No, only the price-mismatch specialist via
  `ResponsesHostServer`. Full handoff hosting is unverified here. The module also deletes its
  temporary version; it is not a persistent production deployment.
- **"Do high groundedness scores prove retrieval worked?"** No. Module 10 compares answers to
  `ground_truth`, not actual retrieved policy context. Capture that context/citations per query
  for strict retrieval-groundedness, and use a representative test set before setting gates.
- **"Does Agent Framework tracing happen automatically?"** It emits native spans, but export
  to Application Insights still needs configuration in the running process. The raw client path
  additionally uses instrumentation and a manual query span.
- **"Can this automatically approve invoices, and is it cheap enough?"** The lab only recommends
  actions; financial authority remains human-owned. Estimate model and judge tokens, extra
  retrieval/handoff calls, Search, Hosted Agent compute, Code Interpreter, and telemetry for real
  traffic. An idle client is not evidence that all cloud charges have stopped.
- **"Can we keep the environment?"** Only with an explicit resource owner, budget, access/retention
  decisions, and a cleanup date. Module 9's temporary version should still be removed immediately;
  preserve shared resources only with the facilitator's approval.
