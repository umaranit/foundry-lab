# Module 7: Multi-agent orchestration with Microsoft Agent Framework

**Objective:** Define three specialized, code-first agents (price mismatch, duplicate invoice,
missing PO) and a coordinator agent using the **Microsoft Agent Framework** SDK, wire them
together with `HandoffBuilder` so the coordinator routes each discrepancy query to exactly one
specialist, and run the resulting workflow against sample queries covering all three paths.

**Estimated duration:** 50 minutes

**Prerequisites:** Modules 5–6 complete (for the `po-records.csv` data and the working Python
environment / `.env` file in your Module 0 working folder). This module does **not** depend on
the hosted **Invoice Discrepancy Assistant** agent itself — only on the data files and local
environment you've already set up.

> **A separate, code-first agent system — not a new version of the hosted agent.** Every module
> so far (2 through 6) built and extended **one** server-hosted Prompt Agent called the
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

1. From your activated virtual environment (the same one from Modules 5–6), install the
   `agent-framework` package:

   ```powershell
   pip install agent-framework --pre
   ```

   > **⚠️ Verify before running:** `agent-framework` is under active development, and package
   > name, extras, and the `--pre` flag requirement can change between releases. Confirm the
   > current install command on the
   > [Microsoft Agent Framework documentation](https://learn.microsoft.com/agent-framework/)
   > or its PyPI page before you run this, and adjust if the command above no longer matches.

2. Confirm you're still signed in with the Azure CLI — this module's agents authenticate with
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
PROJECT_ENDPOINT=https://<your-project-name>.<region>.ai.azure.com/api/projects/<project-name>
MODEL_DEPLOYMENT=invoice-assistant-chat
LAB_REPO_ROOT=C:\path\to\your\local\clone\of\this\lab
```

### 4. Reuse the PO lookup logic with the Agent Framework's `@tool` decorator

Create a new file named `agent_framework_workflow.py` in your working folder. Start with imports
and a re-exposed version of the PO lookup function:

```python
"""Multi-agent invoice discrepancy routing with the Microsoft Agent Framework.

A separate, code-first agent system from the hosted Invoice Discrepancy Assistant built in
Modules 2-6. A coordinator agent classifies each discrepancy query and hands off to exactly one
specialist agent, using agent_framework.HandoffBuilder.
"""

import asyncio
import csv
import os
from pathlib import Path
from typing import Annotated

from agent_framework import Agent, HandoffBuilder, tool
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential
from dotenv import load_dotenv
from pydantic import Field

load_dotenv()

DATA_DIR = Path(os.environ["LAB_REPO_ROOT"]) / "lab" / "data"
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
> for human approval — appropriate here since it's a read-only lookup, in contrast to Module 6's
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
[`data/invoice-po-policy.md`](/lab/data/invoice-po-policy.md) directly in the prompt. This is
necessary because these are plain code-first agents with no File Search / knowledge-base grounding
configured (unlike the hosted agent in Module 3) — if the policy isn't in the instructions, the
agent has no way to know it.

Append to `agent_framework_workflow.py`:

```python
price_mismatch_agent = Agent(
    client=client,
    name="price_mismatch_agent",
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

> **Why does `duplicate_invoice_agent` not get `lookup_po_by_number`?** Per the module scope, the
> duplicate-invoice specialist reasons about invoice-to-invoice matching criteria (vendor,
> amount, invoice number, submission date) — none of which requires a PO lookup. Giving it a tool
> it has no instructions to use would be dead weight; only `price_mismatch_agent` and
> `missing_po_agent` actually need to retrieve a PO record to do their job.

### 7. Define the coordinator agent

Append the coordinator:

```python
coordinator = client.create_agent(
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
    name="coordinator",
)
```

> **Why `client.create_agent(...)` here but `Agent(client=..., ...)` for the specialists?** Both
> are valid ways to construct an Agent Framework agent against the same `client`; the coordinator
> uses the client's `create_agent` convenience method simply to match the official reference
> exercise this module is grounded in. Functionally, `coordinator` is an agent like any other —
> what makes it the "coordinator" is only the role `HandoffBuilder` assigns it in the next step.

### 8. Build the handoff workflow

Append the `HandoffBuilder` wiring:

```python
workflow = (
    HandoffBuilder(
        name="invoice_discrepancy_routing",
        participants=[coordinator, price_mismatch_agent, duplicate_invoice_agent, missing_po_agent],
    )
    .set_coordinator(coordinator)
    .with_interaction_mode("autonomous")  # no mid-run user prompts needed for these sample queries
    .build()
)
```

- `participants` lists every agent in the workflow, including the coordinator itself.
- `set_coordinator(coordinator)` tells `HandoffBuilder` which participant classifies the request
  and hands off — the other three only ever receive a handoff, they never initiate one.
- `with_interaction_mode("autonomous")` runs the whole workflow to completion without pausing for
  a human to approve each handoff. The framework's default interaction mode is a
  human-in-the-loop mode that pauses mid-run for approval — appropriate for a production support
  tool, but more setup than this lab's sample queries need, so we select `"autonomous"` instead.

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
        result = await workflow.run(query)
        # ⚠️ Verify before running: the exact shape of `result` for a HandoffBuilder workflow
        # was not confirmed against an official end-to-end example at the time this module was
        # written. Inspect `result` (for example `print(type(result))`, `result.get_outputs()`
        # if available, or iterating `result` directly) against your installed agent-framework
        # version's docs/IntelliSense, and adjust the line below to match.
        print(f"RESULT: {result}")


if __name__ == "__main__":
    asyncio.run(main())
```

> ## ⚠️ Verify before running
>
> The official Microsoft Agent Framework reference material available at the time this module
> was written showed `HandoffBuilder` workflows driven with `run_stream()`, handling
> `RequestInfoEvent`s for the default human-in-the-loop interaction mode — it did not show a
> confirmed, complete example of what `await workflow.run(query)` returns end-to-end for
> `with_interaction_mode("autonomous")`. Before you run Step 9, check your installed
> `agent-framework` version's docs or VS Code IntelliSense for the `WorkflowRunResult` (or
> equivalent) type `workflow.run()` returns, and adjust the `print(f"RESULT: {result}")` line
> above to correctly extract the final specialist's text response — for example, it may need to
> be `result.get_outputs()`, something iterated from `result`, or a similarly named accessor.

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

- **`ModuleNotFoundError: No module named 'agent_framework'`:** Confirm you ran
  `pip install agent-framework --pre` in the **same** activated virtual environment you're using
  to run the script, and re-check the current install command against the official docs per the
  Step 1 callout — the package name/flags can change between pre-release versions.
- **Authentication errors from `AzureCliCredential`:** Confirm `az login` is still valid
  (`az account show`) — unlike `DefaultAzureCredential` in Modules 5–6, `AzureCliCredential` only
  checks the Azure CLI's signed-in session, so other credential sources (like a managed identity)
  won't help here even if they'd otherwise work.
- **`FileNotFoundError` reading `po-records.csv`:** Confirm `LAB_REPO_ROOT` in `.env` points at
  the root of your local clone of *this* lab repository, as in Module 5.
- **The coordinator hands off to the wrong specialist, or doesn't hand off at all:** Check that
  each specialist's `name=` in `Agent(...)` exactly matches the name the coordinator's
  instructions reference — `HandoffBuilder` matches handoff targets by participant name, so a
  typo or mismatch (for example `price_mismatch_agent` vs. `price-mismatch-agent`) will silently
  fail to route.
- **Confusion about why this doesn't touch the Module 2 hosted agent at all:** That's expected —
  re-read the callout at the top of this module. This is an intentionally separate, code-first
  agent system; nothing here creates a new version of, or otherwise modifies, the **Invoice
  Discrepancy Assistant**.
- **Uncertain what `workflow.run(query)` actually returns:** This is the known unconfirmed detail
  flagged in Step 9's "Verify before running" box — inspect the result object at runtime (for
  example, `print(type(result))` and `print(dir(result))`) against your installed
  `agent-framework` version rather than assuming the shape shown in this module's sample code.
