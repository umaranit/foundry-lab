# Module 10: Wrap-Up & Governance

**Objective:** Review cost awareness and responsible AI considerations for this solution, clean
up the Azure/Foundry resources created during the lab, and point to further learning paths.

**Estimated duration:** 15 minutes

**Prerequisites:** Modules 0–9 complete, or at least attempted — this module is a discussion and
cleanup pass over everything built so far, not new code. You'll need access to the Azure
portal/CLI and the Foundry portal with the permissions you confirmed in Module 0.

## Steps

### 1. Recap what you built

Each module added one capability to the **Invoice Discrepancy Assistant**. Before you tear
anything down, it's worth seeing the whole system in one place:

| # | Module | What you built |
|---|--------|-----------------|
| 0 | [Prerequisites](/lab/modules/00-prerequisites.md) | Confirmed Azure/Foundry access, model quota, VS Code + Foundry Toolkit, Python 3.13, and Git. |
| 1 | [Foundry project & model setup](/lab/modules/01-foundry-project-and-model-setup.md) | Created the `invoice-discrepancy-assistant` Foundry project and deployed the `invoice-assistant-chat` chat model. |
| 2 | [Build the agent (portal)](/lab/modules/02-build-the-agent-portal.md) | Created the **Invoice Discrepancy Assistant** hosted Prompt Agent with system instructions tying its behavior to Contoso's policy. |
| 3 | [Knowledge base / grounding](/lab/modules/03-knowledge-base-grounding.md) | Attached File Search / Foundry IQ so the agent grounds discrepancy guidance in `invoice-po-policy.md` with citations. |
| 4 | [Tools — Built-in (Portal)](/lab/modules/04-tools.md) | Added the Code Interpreter built-in tool and uploaded `discrepancy-trends.csv` for trend analysis. |
| 5 | [Custom Function Tools (Code)](/lab/modules/05-programmatic-dev-vscode-hosted-agent.md) | Added a `lookup_po_by_number` `FunctionTool` backed by `po-records.csv`, created a new agent version, and built a `conversations`/`responses`-based client. |
| 6 | [MCP Integration (Code)](/lab/modules/06-mcp-integration.md) | Built a local FastMCP server exposing an escalation-contact lookup grounded in policy Section 8, and wired it into the client as a `FunctionTool`. |
| 7 | [Multi-agent orchestration (Agent Framework)](/lab/modules/07-multi-agent-orchestration-agent-framework.md) | Built a separate, code-first multi-agent system (classifier + specialists) using the Microsoft Agent Framework, contrasted with the hosted agent from Modules 2–6. |
| 8 | [Evaluation](/lab/modules/08-evaluation.md) | Captured real agent responses and ran a Foundry cloud evaluation with the built-in `intent_resolution`, `relevance`, and `groundedness` evaluators (no custom evaluator). |
| 9 | [Monitoring & tracing](/lab/modules/09-monitoring-and-tracing.md) | Instrumented the agent with Azure Monitor OpenTelemetry and inspected traces/spans for token usage, latency, and tool calls. |

Take a minute to open the Foundry portal and look at the project **Overview** page — you should
be able to point at the model deployment (Module 1), the agent and its tool list (Modules 2–6),
an evaluation run (Module 8), and trace data (Module 9) all in one project.

### 2. Understand what actually costs money

This lab is small, but it's worth knowing exactly which pieces of it carry ongoing cost versus a
one-time or negligible cost, so you can reason about a larger, production-scale version of this
same system.

| Resource | Created in | Cost pattern |
|---|---|---|
| Foundry project itself | Module 1 | No direct cost; it's a management container. |
| `invoice-assistant-chat` model deployment | Module 1 | **Ongoing.** A Standard (pay-as-you-go) deployment bills per input/output token on every call — from the playground, your Module 5/6 client, and the Module 7 Agent Framework agents alike. This is almost always the largest cost driver in a real deployment, since every test query, evaluation row, and multi-agent hop consumes tokens. |
| Stored policy file + File Search vector store | Module 3 | **One-time/negligible** for a single small Markdown document. Storage and indexing cost scale with document count/size, not query volume, so this is cheap at lab scale but worth watching if you attach a larger real policy corpus. |
| `discrepancy-trends.csv` uploaded to Code Interpreter | Module 4 | **One-time/negligible.** A small CSV costs very little to store; Code Interpreter *session* time (the sandbox that runs generated Python) is metered separately but is brief and infrequent in this lab. |
| Application Insights ingestion | Module 9 | **Ongoing**, and easy to underestimate. Every traced span — every model call, every tool call, every test query you sent in Module 9 — is ingested and billed by data volume. A production agent handling real invoice volume would generate meaningfully more telemetry than this lab did. |
| Evaluation run data | Module 8 | **One-time/negligible** for the size of dataset used in this lab. The cloud evaluation run itself also consumes model tokens (the model under test *and* the judge model invoked by each evaluator), so cost scales with dataset size and how often you re-run evaluations, not with storage. |

> **Note:** The Module 7 Agent Framework agents (classifier + specialists) don't add a *separate*
> Azure resource or cost line — they're code-first agents that call the same
> `invoice-assistant-chat` model deployment from Module 1 directly. The cost impact of Module 7
> is simply that a single user question can now fan out into multiple model calls (classify,
> then route to a specialist), which is worth knowing before you scale a multi-agent pattern like
> this to production traffic.

The two lines to actually watch if you kept this running past the workshop are the **model
deployment** and **Application Insights ingestion** — both scale with usage, not with how much
you built. Everything else in this lab is effectively a fixed, small cost regardless of how long
it sits around.

### 3. Responsible AI considerations for this scenario

This lab is a toy, but the pattern it demonstrates — an LLM-based agent making recommendations
about financial approvals — raises real responsible-AI questions that are worth calling out
explicitly, grounded in what you actually built:

- **Never let the agent bypass the human-approval gates it was built to respect.** Policy
  Section 2 (general triage workflow) and Section 7 (approval authority limits) in
  `invoice-po-policy.md` already require supervisor sign-off, Procurement escalation, or
  Controller sign-off above specific dollar/percentage thresholds — and Section 7 is explicit
  that *"an analyst may never approve a discrepancy resolution that exceeds their own authority
  limit, even if the discrepancy itself ... falls within an auto-approve threshold elsewhere in
  this policy."* The same principle applies to this agent: it can draft a recommendation, explain
  the applicable threshold, and even summarize what a response to the vendor could say, but it
  must never be wired up to *automatically* approve, reject, or short-pay an invoice on behalf of
  a human approver once a question crosses one of those thresholds — even though nothing
  technically stops you from calling the model's output directly into a payment action. The
  agent's job in this lab is triage and recommendation; the sign-off gates in the policy exist
  specifically so a human stays accountable for every dollar released above the analyst's own
  $2,500 auto-approve ceiling.
- **Guard against hallucinated PO or policy data.** An agent that invents a PO number, a unit
  price, or a tolerance threshold is worse than one that says "I don't know" — a confidently wrong
  answer here could lead to a real overpayment or an incorrectly rejected invoice. This is exactly
  why Module 2's system instructions explicitly forbid inventing a threshold or policy detail, why
  Module 3 grounds policy answers in the actual `invoice-po-policy.md` text with citations instead
  of letting the model answer from parametric memory, and why Modules 5–6 replace "ask the model
  to guess a PO's price/quantity" and "ask the model to guess an escalation contact" with real
  tool calls (`lookup_po_by_number` against `po-records.csv`, `get_escalation_contact` against
  policy Section 8) that return actual data. Every layer you added after Module 2 was, in part, a
  defense against hallucination for a specific category of fact.
- **Treat PO and discrepancy data as financial/vendor records, even though it's fictional here.**
  `po-records.csv` and `discrepancy-trends.csv` contain vendor names, unit prices, quantities, and
  dollar amounts — the kind of data a real AP system would classify as confidential vendor and
  financial information (and in some organizations, data with contractual confidentiality
  obligations to the vendor). Nothing in this lab required real PII, and none was used, but if you
  adapt this pattern to real company data, apply the same access controls, retention policies, and
  least-privilege principles to the PO/discrepancy records a tool can read as you would to any
  other financial system — and remember that Module 9's tracing captures tool inputs/outputs
  (including PO numbers and amounts) in Application Insights, so a production deployment should
  review retention and access settings on that telemetry too, not just on the source CSVs.

### 4. Clean up resources

Delete resources in this order so dependent resources don't block deletion, then take the single
step that guarantees nothing is left behind.

1. **Delete the model deployment (Module 1).** In the Foundry portal, go to **Models +
   endpoints**, select the `invoice-assistant-chat` deployment, and select **Delete**. This stops
   the ongoing per-token billing immediately.
2. **Delete the agent (Module 2).** In the Foundry portal, go to **Agents**, select the
   **Invoice Discrepancy Assistant**, and select **Delete**. This removes the agent and its
   versions (including the tool configuration added in Modules 3–6).
3. **Delete the resource group (covers everything else in one step).** Rather than hunting down
   the Application Insights resource, the File Search vector store, the Code Interpreter upload,
   and the project itself individually, delete the whole resource group you created in Module 1:

   ```powershell
   az group delete --name rg-invoice-discrepancy-assistant --yes --no-wait
   ```

   Deleting the resource group is simpler *and* more complete than deleting resources one at a
   time — it removes the Foundry project, the connected Application Insights resource (and all
   ingested trace data from Module 9), every stored file and vector store (Module 3), the Code
   Interpreter upload (Module 4), and the evaluation run data (Module 8) in a single operation,
   with no risk of missing a resource that was created implicitly on your behalf. Replace
   `rg-invoice-discrepancy-assistant` with your actual resource group name if you used a different one
   or a facilitator-provided shared group (in which case, confirm with your facilitator before
   deleting it — a shared group may still be in use by other participants).
4. **Stop any local processes (no Azure cost, but good hygiene).** If you still have a terminal
   running the Module 6 `server.py` MCP server, or a `client.py` session from Modules 5–7, stop
   them with `Ctrl+C`. These run entirely on your machine and never had an Azure cost of their
   own, but there's no reason to leave them running after the lab.

### 5. Where to go next

This lab deliberately kept each module small enough to fit a workshop. To go deeper on either
half of what you built:

- **[Develop AI agents on Azure](https://learn.microsoft.com/training/paths/develop-ai-agents-on-azure/)**
  — the full Microsoft Learn path behind Modules 0–7 (hosted agents, grounding, tools, and
  multi-agent orchestration).
- **[Operationalize generative AI apps](https://learn.microsoft.com/training/paths/operationalize-gen-ai-apps/)**
  — the full Microsoft Learn path behind Modules 8–9 (evaluation and monitoring/tracing).

If you want to keep experimenting on your own before moving on, two ideas earlier modules
flagged but intentionally didn't implement are worth a look:

- Module 6's note that this lab only covers **local, stdio-based** MCP servers — explore the
  **approval-loop pattern for remote MCP servers** (the `MCPTool`/approval-request flow mentioned
  but set aside in that module) if your own scenario needs to call a hosted MCP endpoint over
  HTTP instead of a subprocess you control.
- Module 9's mention of a deeper **local trace-tree script** (`check_traces.py` in the
  `mslearn-genaiops` reference repo) that queries Log Analytics directly and prints a nested trace
  tree in the terminal — useful if the Foundry portal's Traces view isn't enough detail for your
  debugging workflow.

## ✅ Checkpoint / Validate

Before you consider this lab complete, confirm:

- [ ] You can explain, in one sentence each, what Modules 0–9 added to the system.
- [ ] You can name which of this lab's resources incur *ongoing* cost (model deployment,
  Application Insights ingestion) versus one-time/negligible cost (stored files, evaluation data).
- [ ] You can explain why an AP automation agent must never auto-approve past the thresholds in
  policy Section 2/7, even though nothing technically prevents it.
- [ ] You have either deleted the model deployment, agent, and resource group, or you have
  explicitly confirmed with your facilitator that a shared resource group should stay up.
- [ ] Any local `server.py` or `client.py` processes from Modules 5–7 are stopped.

## 🚀 Try it yourself

Sketch (on paper, in a doc, or as a quick diagram) how you'd extend this Invoice Discrepancy
Assistant pattern for your own organization's accounts-payable workflow. Consider: What would
replace `invoice-po-policy.md` — an existing policy document, or one you'd need to write first?
What system would `lookup_po_by_number` need to call instead of a CSV — an ERP API, a database?
Which approval gates from your own policy would need to stay strictly human-owned, the way
Section 7's authority limits do here? You don't need to build any of it — the goal is just to see
how directly Modules 0–9 map onto a real system.
