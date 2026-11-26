# Facilitator reference: Modules 1–11 — what we did and why

**Opening question:** An invoice says $15.40 per bracket, but the purchase order (PO) says
$14.80. What should the accounts-payable analyst do?

A general-purpose model can offer plausible advice. To give advice the analyst can rely on, the
assistant needs the company's policy, the actual PO, the quantity affected, and the person
responsible for the decision. The workshop first gives the assistant those capabilities, then
explores where its code runs and how to tell whether its answers are good.

## Modules 1–7: Give one assistant knowledge and tools

| Module | What participants do | Why it matters |
|---|---|---|
| [1. Project and model](modules/01-foundry-project-and-model-setup.md) | Create a Foundry project, deploy a chat model, and send a test prompt. | A reply proves the model is reachable. It does **not** mean the model knows Contoso's policy or can look up an invoice. |
| [2. Build the Prompt Agent](modules/02-build-the-agent-portal.md) | Give the Invoice Discrepancy Assistant a role, instructions, and limits. | Instructions define **what job to do**. They don't supply the company's rules or transaction records. |
| [3. Ground answers in policy](modules/03-knowledge-base-grounding.md) | Upload the invoice policy and use File Search to retrieve relevant passages. | “What variance can I approve?” calls for an answer from **Contoso's policy**, not a guess about what companies usually allow. |
| [4. Analyze historical data](modules/04-tools.md) | Give the agent Code Interpreter and historical discrepancy data. | “Which vendor drove the increase?” requires grouping and calculating over records. Searching a policy paragraph won't produce that answer. |
| [5. Look up a purchase order](modules/05-programmatic-dev-vscode-hosted-agent.md) | Write a Python function that retrieves a PO from the lab CSV. The model requests the lookup; the local client executes it and returns the result. | The policy tells us **which rule applies**. The PO tells us **which price was agreed**. Without the lookup, the assistant has no basis for claiming the price was $14.80. |
| [6. Connect a tool through MCP](modules/06-mcp-integration.md) | Expose an escalation-contact lookup through a separate local Model Context Protocol (MCP) server. The client discovers and calls the tool. | Once the discrepancy is classified, the analyst needs to know **who handles it and by when**. A standard tool interface makes the lookup available without embedding its implementation in each client. |
| [7. Compare knowledge retrieval](modules/07-foundry-iq-knowledge-base.md) | Add a Foundry IQ knowledge base alongside File Search. Compare evidence and citations for the same policy and approve or deny a tool request. | The point isn't to add a second policy. It's to compare **how the agent obtains evidence** and see where a person can control access. |

**Teaching cue:** Instructions define the job; documents supply the rules; tools retrieve facts
or perform calculations. Ask which of those three could establish the price on a specific PO.

## Modules 8–9: Try a different design, then host one specialist

**Transition:** Participants start a separate code-first implementation. The Prompt Agent from
the earlier modules has not turned into a multi-agent system.

| Module | What participants do | Why it matters |
|---|---|---|
| [8. Route work to specialists](modules/08-multi-agent-orchestration-agent-framework.md) | Build a local Agent Framework workflow. A coordinator routes a case to a price-mismatch, duplicate-invoice, or missing-PO specialist. | The design question changes from “How should one assistant answer?” to **“Who should handle this case?”** A missing PO shouldn't go through a price comparison. |
| [9. Deploy one specialist](modules/09-deploy-to-foundry-hosted-agents.md) | Package only the price-mismatch specialist, its dependencies, and its PO data. Invoke it in Foundry, inspect the trace, then clean up the temporary version. | The specialist and its lookup now run **without the laptop executing their Python code**. Cleanup restores prior routing where applicable and stops the temporary version's compute charges. It doesn't undo the demonstration or remove the original Prompt Agent. |

**Return to the opening invoice:** $15.40 minus $14.80 is $0.60 per bracket. Across 1,000
brackets, the impact is **$600**. The percentage is about **4.05%**, but the dollar impact
exceeds the $250 threshold. The higher tier wins: hold payment on the disputed line and
escalate to Procurement, with a response due within three business days. This is a
recommendation; the assistant hasn't changed a payment or opened a case.

## Modules 10–11: Judge the answers and inspect execution

**Transition:** The capture and tracing exercises return to the **Module 6 Prompt Agent**.
They do not require the temporary Module 9 specialist to remain deployed.

| Module | What participants do | Why it matters |
|---|---|---|
| [10. Evaluate answer quality](modules/10-evaluation.md) | Capture the agent's real answers to 15 known questions. Score the saved answers for intent resolution, relevance, and groundedness, then inspect an individual result. | A convincing demo isn't a quality test. In one workshop run, an answer earned **4/5 for groundedness** because it quoted correct approval limits that the reference answer didn't include. Reading the answer, reference, and evaluator's explanation distinguishes a gap in the reference from a mistake by the agent. Scores can vary between runs. |
| [11. Monitor and trace execution](modules/11-monitoring-and-tracing.md) | Instrument the Python client, send sample queries, inspect project usage, and follow an individual query's trace. | **Monitoring** asks “How much activity are we seeing overall?” **Tracing** asks “What happened during this particular request?” A project's token total can't tell you how long one invoice lookup took. Neither a usage chart nor a successful trace proves that the financial advice was correct. |

**Closing takeaway:** Modules 1–7 build capability. Modules 8–9 explore design and deployment.
Modules 10–11 gather evidence about quality and execution. Throughout, the assistant
recommends an action; the authorized human makes the financial decision.
