# Module 8: Multi-agent orchestration with Microsoft Agent Framework

**Objective:** Define three specialized, code-first agents (price mismatch, duplicate invoice,
missing PO) and a coordinator agent using the **Microsoft Agent Framework** SDK, wire them
together with `HandoffBuilder` so the coordinator routes each discrepancy query to exactly one
specialist, and run the resulting workflow against sample queries covering all three paths.

**Estimated duration:** 50 minutes

**Prerequisites:** Modules 5–6 complete (for the `po-records.csv` data and the working Python
environment / `.env` file in your Module 0 working folder). This module does **not** depend on
the **Invoice Discrepancy Assistant** Prompt Agent itself — only on the data files and local
environment you've already set up.

## Why this module?

- **Where we are:** Modules 2–7 explored one Prompt Agent with knowledge and tools.
- **What's missing:** We have not explored explicit routing between specialists with separate
  responsibilities. Adding another retrieval tool does not teach that orchestration pattern.
- **What we'll add:** A separate Agent Framework system: a coordinator routes sample requests
  to a price-mismatch, duplicate-invoice, or missing-PO specialist. This is an architectural
  alternative, not a required upgrade or another version of the earlier Prompt Agent.
- **Where it runs:** Agent orchestration and the CSV lookup run in local Python; model calls
  go to Foundry. These agents do not inherit the Prompt Agent's tools or knowledge sources:
  this exercise embeds policy in specialist instructions instead.
- **By the end:** You can run sample queries for all three paths and observe the intended
  specialist handoff. You have a working local workflow, not a deployed multi-agent service.

> **How this relates to a Hosted Agent:** This module runs the coordinator, specialists, and
> PO lookup locally; only the model calls go to Foundry. In
> [Module 9](09-deploy-to-foundry-hosted-agents.md), you adapt the price-mismatch specialist
> and its PO lookup into a Foundry Hosted Agent that runs in a managed container. Module 9 does
> **not** deploy the coordinator, the other specialists, or the full multi-agent workflow. The
> Prompt Agent from Modules 2–7 remains separate.

> **A separate, code-first agent system — not a new version of the Prompt Agent.** Every module
> so far (2 through 7) built and extended **one** Prompt Agent called the
> **Invoice Discrepancy Assistant**, configured first in the portal and then given new versions
> from code. This module is different: you are going to define **brand-new agents entirely in
> code**, using the Microsoft Agent Framework SDK, that exist only as Python objects in your
> script — they are not versions of the Invoice Discrepancy Assistant, they don't share its
> agent ID, and nothing you do here changes it. Think of this module as building a **second,
> independent agent system** that explores a different architecture (several small, specialized
> agents coordinated by a router) for the same scenario. Microsoft recommends the Agent Framework
> as the code-first orchestration path going forward, in place of the now-retiring Prompt Flow.

## Steps

### 1. Install the Microsoft Agent Framework SDK

1. In the VS Code terminal, go to your working folder and activate its virtual environment
   (the same one from Modules 5–6). Your prompt must start with `(.venv)` before you continue:

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   > Activate the `.venv` **inside the folder where you'll run the script**. Activating a `.venv`
   > from a different folder gives you a different Python, and the import will fail.

2. Install the `agent-framework` packages using `python -m pip` so they go into the active
   environment, then verify the import:

   ```powershell
   python -m pip install agent-framework-core==1.14.0 agent-framework-orchestrations==1.1.0 agent-framework-foundry==1.11.0 azure-identity python-dotenv
   python -c "from agent_framework.foundry import FoundryChatClient; from agent_framework.orchestrations import HandoffBuilder; print('agent_framework OK')"
   ```

   > Install these three packages, **not** the `agent-framework` meta-package. The meta-package
   > installs every integration (`agent-framework-core[all]`), and one of them
   > (`agent-framework-azurefunctions`) now requires a newer core version, so pip fails with
   > `ResolutionImpossible`. This module needs only core, orchestrations, and the Foundry client.

3. Confirm you're still signed in with the Azure CLI — this module's agents authenticate with
   `AzureCliCredential` rather than the `DefaultAzureCredential` used in Modules 5–6 (both work;
   this just matches the convention used in the Agent Framework's own reference samples):

   ```powershell
   az account show --output table
   ```

### 2. Understand why this module uses different building blocks than Modules 5–6

Before writing code, it's worth being explicit about three things that look similar to what you
did in Modules 5–6 but are not the same:

1. **Tool definitions.** Modules 5–6 defined tools as `azure.ai.projects.models.FunctionTool`
   objects, hand-building a JSON schema for the parameters. The Agent Framework has its own,
   simpler idiom: decorate a plain Python function with `@tool`, and describe each parameter with
   a `typing.Annotated` type hint and a `pydantic.Field`. The framework derives the schema from
   the type hints for you — you will not write a JSON schema by hand in this module.
2. **Orchestration pattern.** You may have seen (or will see elsewhere in the Agent Framework
   ecosystem) a `SequentialBuilder`, which runs every participant agent in a fixed pipeline, one
   after another, passing each agent's output to the next. That pattern fits a scenario like
   "draft, then review, then finalize." It does **not** fit this scenario, because an invoice
   discrepancy is always **exactly one** of price mismatch, duplicate invoice, or missing PO — you
   want a router that reads the query, decides which single specialist applies, and hands off to
   **only that one** agent. That's what `HandoffBuilder` does: one coordinator agent classifies
   the request and hands off control to one matching specialist, rather than running all of them.
3. **Credential type.** `AzureCliCredential` instead of `DefaultAzureCredential`. Both work for a
   developer signed in with `az login`; this module simply follows the convention used in the
   Agent Framework's own reference exercises.

### 3. Configure environment variables

If you still have the `.env` file from Module 5 in your working folder, you can reuse it as-is —
this module reads the same `PROJECT_ENDPOINT`, `MODEL_DEPLOYMENT`, and `LAB_REPO_ROOT` values.
Confirm your `.env` contains:

```text
PROJECT_ENDPOINT=https://<resource-name>.services.ai.azure.com/api/projects/<project-name>
MODEL_DEPLOYMENT=invoice-assistant-chat
LAB_REPO_ROOT=C:\path\to\your\local\clone
```

Use the complete **Project endpoint** copied from your project's Overview page. Do not construct
the URL from the project name or substitute the separate Azure OpenAI endpoint.

### 4. Reuse the PO lookup logic with the Agent Framework's `@tool` decorator

Create a new file named `agent_framework_workflow.py` in your working folder. Start with imports
and a re-exposed version of the PO lookup function:

```python
"""Multi-agent invoice discrepancy routing with the Microsoft Agent Framework.

A separate, code-first agent system from the Invoice Discrepancy Assistant Prompt Agent built in
Modules 2-7. A coordinator agent classifies each discrepancy query and hands off to exactly one
specialist agent, using agent_framework.HandoffBuilder.
"""

import asyncio
import csv
import os
from pathlib import Path
from typing import Annotated

from agent_framework import Agent, tool
from agent_framework.orchestrations import HandoffBuilder
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential
from dotenv import load_dotenv
from pydantic import Field

load_dotenv()

DATA_DIR = Path(os.environ["LAB_REPO_ROOT"]) / "data"
PO_RECORDS_FILE = DATA_DIR / "po-records.csv"


@tool(approval_mode="never_require")
def lookup_po_by_number(
    po_number: Annotated[str, Field(description="The PO number to look up, e.g. 'PO-10234'")]
) -> str:
    """Look up a purchase order record by its PO number."""
    normalized = po_number.strip().upper()

    with PO_RECORDS_FILE.open(newline="", encoding="utf-8") as csv_file:
        reader = csv.DictReader(csv_file)
        for row in reader:
            if row["po_number"].strip().upper() == normalized:
                return (
                    f"Found PO {row['po_number']}: vendor={row['vendor_name']}, "
                    f"item={row['item_description']}, unit_price={row['unit_price']}, "
                    f"quantity={row['quantity']}, order_date={row['order_date']}, "
                    f"status={row['status']}"
                )

    return (
        f"No purchase order found matching '{po_number}'. Treat this as a missing-PO case "
        "per policy Section 6 rather than guessing a result."
    )
```

> **Same CSV-reading logic as Module 5, different exposure.** The lookup logic (normalize, scan
> `po-records.csv`, return a record or a "not found" message) is the same idea as
> `functions.py`'s `lookup_po_by_number` from Module 5. What changed is *only* how it's exposed as
> a tool: `@tool(approval_mode="never_require")` plus `Annotated[..., Field(description=...)]`
> instead of a hand-written `FunctionTool(..., parameters={...})` JSON schema. `
> approval_mode="never_require"` tells the framework this tool is safe to call without pausing
> for human approval — appropriate here since it's a read-only lookup, in contrast to Module 7's
> MCP tool-approval loop for a different reason (a remote-style approval pattern that doesn't
> apply to this local tool either).

### 5. Create the Foundry chat client

Append the client setup to `agent_framework_workflow.py`:

```python
client = FoundryChatClient(
    project_endpoint=os.environ["PROJECT_ENDPOINT"],
    model=os.environ["MODEL_DEPLOYMENT"],
    credential=AzureCliCredential(),
)
```

This `FoundryChatClient` is how Agent Framework agents call your Foundry project's deployed
model — it plays a similar role to the `AIProjectClient`/`openai_client` pair in Modules 5–6, but
it's a different client class belonging to the Agent Framework SDK.

### 6. Define the three specialist agents

Each specialist's instructions embed the actual tolerance thresholds from the relevant section of
[`data/invoice-po-policy.md`](../data/invoice-po-policy.md) directly in the prompt. This is
necessary because these are plain code-first agents with no File Search / knowledge-base grounding
configured (unlike the Prompt Agent in Module 3) — if the policy isn't in the instructions, the
agent has no way to know it.

Append to `agent_framework_workflow.py`:

```python
price_mismatch_agent = Agent(
    client=client,
    name="price_mismatch_agent",
    require_per_service_call_history_persistence=True,
    instructions="""
You are a specialist in invoice price-mismatch discrepancies for Contoso Finance accounts
payable, per policy Section 3 (CF-AP-POL-014).

A price mismatch is when the invoiced unit price differs from the PO unit price. Use the
lookup_po_by_number tool to retrieve the authoritative PO unit price before comparing - never
guess a PO's price.

Apply these tolerance thresholds (evaluate both the percentage AND the dollar impact; if either
is exceeded, use the higher tier):
- <= 2% variance AND <= $25 total line-item impact: Auto-approve. The analyst may approve the
  invoice at the invoiced amount without further review.
- > 2% but <= 5% variance, OR > $25 but <= $250 total line-item impact: Supervisor sign-off
  required. Route to the AP supervisor queue. Supervisor must respond within 2 business days.
- > 5% variance OR > $250 total line-item impact: Escalate to Procurement. Hold payment and open
  a price-discrepancy case with the Procurement category manager. Procurement must respond within
  3 business days. Do not pay any portion of the disputed line until Procurement confirms the
  correct price.

Report the discrepancy's dollar/percentage variance, which tier it falls into, and the required
next action and SLA.
""",
    tools=[lookup_po_by_number],
)

duplicate_invoice_agent = Agent(
    client=client,
    name="duplicate_invoice_agent",
    require_per_service_call_history_persistence=True,
    instructions="""
You are a specialist in duplicate-invoice discrepancies for Contoso Finance accounts payable, per
policy Section 5 (CF-AP-POL-014).

Treat an invoice as a probable duplicate if it matches an existing invoice on ALL of: same
vendor; same PO number (or same vendor + same invoice total if no PO is referenced); invoice
total within $1.00 of an existing invoice; and submitted within a 60-calendar-day window of the
existing invoice's submission date.

An exact match on the vendor's invoice number against an existing record is always a confirmed
duplicate, regardless of the 60-day window, unless the vendor confirms in writing the invoice
number was mistakenly reused.

Resolution:
- Confirmed duplicate: reject immediately, do not route for approval, notify the vendor's AP
  contact with the original invoice's reference number and payment status.
- Probable duplicate: place on hold and contact the vendor within 2 business days to confirm
  whether it's a resubmission, billing error, or legitimately separate charge.
- If the vendor confirms it is a duplicate, reject using the confirmed-duplicate process.
- If the original invoice was already paid, also report it to the AP supervisor as a potential
  overpayment.

State clearly whether the case is a confirmed duplicate or a probable duplicate, and the
corresponding next action.
""",
)

missing_po_agent = Agent(
    client=client,
    name="missing_po_agent",
    require_per_service_call_history_persistence=True,
    instructions="""
You are a specialist in missing-PO discrepancies for Contoso Finance accounts payable, per policy
Section 6 (CF-AP-POL-014).

A missing-PO condition exists when an invoice references a PO number that does not exist in the
PO system, or has no PO number at all. Use the lookup_po_by_number tool to confirm the PO truly
does not exist before treating it as missing - check for a transposed-digit typo or an expired/
closed PO first (if a closed/expired PO otherwise matches, route to Procurement about reopening
it rather than treating it as a true missing-PO case).

Once confirmed missing, apply these thresholds based on invoice total:
- <= $500: the analyst may request a retroactive PO directly from the requisitioner's manager.
- > $500 and <= $5,000: a retroactive PO requires sign-off from both the requisitioner's manager
  AND the AP supervisor.
- > $5,000: retroactive POs are not permitted. Reject the invoice and return it to the vendor
  with instructions to obtain a valid PO before resubmission.

Report whether the PO is truly missing (vs. a typo or closed/expired PO), the applicable
threshold tier, and the required next action.
""",
    tools=[lookup_po_by_number],
)
```

> **Why `require_per_service_call_history_persistence=True`?** `HandoffBuilder` requires this on
> every agent in a handoff workflow. Without it, `.build()` fails with a `ValueError`. It keeps
> each agent's local conversation history in sync with the model service during handoffs.

> **Why does `duplicate_invoice_agent` not get `lookup_po_by_number`?** Per the module scope, the
> duplicate-invoice specialist reasons about invoice-to-invoice matching criteria (vendor,
> amount, invoice number, submission date) — none of which requires a PO lookup. Giving it a tool
> it has no instructions to use would be dead weight; only `price_mismatch_agent` and
> `missing_po_agent` actually need to retrieve a PO record to do their job.

### 7. Define the coordinator agent

Append the coordinator:

```python
coordinator = Agent(
    client=client,
    name="coordinator",
    require_per_service_call_history_persistence=True,
    instructions=(
        "You are the routing coordinator for Contoso Finance's invoice discrepancy triage "
        "system. Read the analyst's discrepancy description and decide which ONE specialist "
        "should handle it:\n"
        "- 'price_mismatch_agent': the invoiced unit price differs from the PO unit price.\n"
        "- 'duplicate_invoice_agent': the invoice appears to duplicate one already submitted "
        "or paid.\n"
        "- 'missing_po_agent': the invoice references a PO number that can't be found, or no "
        "PO number was given at all.\n"
        "Call the matching handoff tool for exactly one specialist - do not try to resolve the "
        "discrepancy yourself."
    ),
)
```

### 8. Build the handoff workflow

Append the `HandoffBuilder` wiring:

```python
SPECIALIST_NAMES = {"price_mismatch_agent", "duplicate_invoice_agent", "missing_po_agent"}


def create_workflow():
    return (
        HandoffBuilder(
            name="invoice_discrepancy_routing",
            participants=[coordinator, price_mismatch_agent, duplicate_invoice_agent, missing_po_agent],
        )
        .with_start_agent(coordinator)
        .add_handoff(coordinator, [price_mismatch_agent, duplicate_invoice_agent, missing_po_agent])
        .with_autonomous_mode(agents=[coordinator], turn_limits={"coordinator": 1})
        .with_termination_condition(
            lambda messages: any(
                message.role == "assistant"
                and message.author_name in SPECIALIST_NAMES
                and message.text.strip()
                for message in messages
            )
        )
        .build()
    )
```

- `participants` lists every agent in the workflow, including the coordinator itself.
- `with_start_agent(coordinator)` starts the workflow with the coordinator, which classifies the
  request.
- `add_handoff(coordinator, [...])` defines the coordinator's handoff targets: the three
  specialists that can handle the request.
- Autonomous mode lets the coordinator retry once if it doesn't hand off. The termination condition
  stops the workflow when a specialist produces an answer, instead of repeatedly prompting it.
- `create_workflow()` creates a fresh workflow for each independent query.

### 9. Run the workflow against sample queries

Append a small async driver that runs three sample queries, one per specialist path:

```python
SAMPLE_QUERIES = [
    "Invoice for PO-10234 bills 1000 units at $16.50/unit, but the PO says $14.80/unit. "
    "What should I do?",
    "We received an invoice from Fabrikam Industrial Supply, invoice #INV-88213, for $4,200. "
    "We already paid invoice #INV-88213 from Fabrikam three weeks ago for the same amount. "
    "Is this a duplicate?",
    "An invoice for $1,200 references PO-10243, but I can't find that PO number anywhere in "
    "the system. What's the next step?",
]


async def main() -> None:
    for query in SAMPLE_QUERIES:
        print(f"\nQUERY: {query}")
        workflow = create_workflow()
        result = await workflow.run(query)
        answers = [
            output
            for output in result.get_outputs()
            if output.text and any(
                message.role == "assistant" and message.author_name in SPECIALIST_NAMES
                for message in output.messages
            )
        ]
        if not answers:
            raise RuntimeError("The workflow did not produce a specialist answer.")
        print(f"RESULT: {answers[-1].text}")


if __name__ == "__main__":
    asyncio.run(main())
```

`result.get_outputs()` returns the agent responses emitted during the workflow. The driver prints
the specialist's answer and reports an error if no specialist answered.

### 10. Run it and confirm routing

1. From your activated virtual environment, run:

   ```powershell
   python agent_framework_workflow.py
   ```

2. Confirm all three queries run without errors and each produces a result:
   - **Query 1** (PO-10234 price variance) should route to `price_mismatch_agent`, which calls
     `lookup_po_by_number`, confirms the PO price is $14.80, calculates that $16.50 is roughly
     11.5% over and well past the $250 line-item impact on 1000 units, and reports this must be
     **escalated to Procurement** per Section 3.1's Tier 3 threshold.
   - **Query 2** (Fabrikam duplicate invoice number) should route to `duplicate_invoice_agent`,
     which recognizes the exact invoice-number match (`INV-88213` seen twice) as a **confirmed
     duplicate** per Section 5.1, and reports it should be rejected immediately with the vendor's
     AP contact notified — and, since the original was already paid, also flagged to the AP
     supervisor as a potential overpayment per Section 5.2.
   - **Query 3** (PO-10243, a gap in `po-records.csv`) should route to `missing_po_agent`, which
     calls `lookup_po_by_number`, confirms no record matches `PO-10243`, and — since the invoice
     total of $1,200 is over $500 and under $5,000 per Section 6.1 — reports that a retroactive PO
     requires sign-off from both the requisitioner's manager **and** the AP supervisor.

## ✅ Checkpoint / Validate

- [ ] `python agent_framework_workflow.py` runs without import or authentication errors.
- [ ] Query 1 (price variance on PO-10234) is routed to `price_mismatch_agent` and the result
  correctly cites the Section 3.1 Tier 3 ("escalate to Procurement") threshold.
- [ ] Query 2 (repeated Fabrikam invoice number) is routed to `duplicate_invoice_agent` and the
  result correctly identifies it as a **confirmed** duplicate (exact invoice-number match), not
  merely a probable one.
- [ ] Query 3 (PO-10243) is routed to `missing_po_agent`, which calls `lookup_po_by_number`,
  confirms no match, and cites the correct Section 6.1 dollar-threshold tier for a $1,200 invoice.
- [ ] You can explain, in your own words, why this module uses `HandoffBuilder` instead of a
  `SequentialBuilder`, and why `@tool`/`Annotated`/`Field` replaces the `FunctionTool`/JSON-schema
  approach from Modules 5–6.

## 🚀 Try it yourself

Add a fourth sample query that's deliberately ambiguous between two specialists — for example, an
invoice that both references a PO your coordinator can't immediately confirm exists *and* has a
price that looks off. Run it through the workflow and observe which single specialist the
coordinator picks. Then try sharpening the coordinator's instructions (for example, telling it to
prefer confirming the PO exists before reasoning about price) and re-run the same query to see if
the routing decision changes.

## ⚠️ Troubleshooting

- **`ModuleNotFoundError: No module named 'agent_framework'`:** The packages were installed into
  a different Python than the one running the script. This usually happens if you ran `pip install`
  before activating `.venv`, or activated a `.venv` from another folder. With `(.venv)` showing,
  run the Step 1 install command again with `python -m pip`, then run the script again.
- **`ResolutionImpossible` mentioning `agent-framework-azurefunctions` or `agent-framework-core`:**
  You installed the `agent-framework` meta-package. Run the Step 1 command instead, which installs
  only `agent-framework-core`, `agent-framework-orchestrations`, and `agent-framework-foundry`.
- **`ValueError: Handoff workflows require all participant agents to have
  'require_per_service_call_history_persistence=True'`:** Every `Agent(...)` in the workflow
  (the coordinator and all three specialists) must include
  `require_per_service_call_history_persistence=True`, as shown in Steps 6 and 7.
- **Warnings like `No handoff configuration found for agent 'price_mismatch_agent'`:** These are
  expected. The specialists are the final step, so they don't hand off further.
- **Authentication errors from `AzureCliCredential`:** Confirm `az login` is still valid
  (`az account show`) — unlike `DefaultAzureCredential` in Modules 5–6, `AzureCliCredential` only
  checks the Azure CLI's signed-in session, so other credential sources (like a managed identity)
  won't help here even if they'd otherwise work. If the Azure CLI token request times out after
  10 seconds, check that `az account get-access-token --resource "https://ai.azure.com" --query
  expiresOn -o tsv` succeeds. If it succeeds but is slow, use
  `credential=AzureCliCredential(process_timeout=60)` in Step 5.
- **The workflow keeps running after a specialist answers:** Check that the Step 8 workflow
  includes the termination condition, and that Step 9 creates a fresh workflow for each query.
  If it still waits, investigate authentication and model connectivity separately.
- **`FileNotFoundError` reading `po-records.csv`:** Confirm `LAB_REPO_ROOT` in `.env` points at
  the root of your local clone of this repository (the folder directly containing
  `data/po-records.csv`), as in Module 5.
- **The coordinator hands off to the wrong specialist, or doesn't hand off at all:** Check that
  each specialist's `name=` in `Agent(...)` exactly matches the name the coordinator's
  instructions reference — `HandoffBuilder` matches handoff targets by participant name, so a
  typo or mismatch (for example `price_mismatch_agent` vs. `price-mismatch-agent`) will silently
  fail to route.
- **Confusion about why this doesn't touch the Module 2 Prompt Agent at all:** That's expected —
  re-read the callout at the top of this module. This is an intentionally separate, code-first
  agent system; nothing here creates a new version of, or otherwise modifies, the **Invoice
  Discrepancy Assistant**.
