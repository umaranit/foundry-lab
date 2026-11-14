# Module 5: Custom Function Tools (Code)

**Objective:** Define a `FunctionTool` for "look up PO by number" (backed by `po-records.csv`),
create a new version of the **Invoice Discrepancy Assistant** Prompt Agent in code with
`project_client.agents.create_version`, and build a client that calls it using
`openai_client.conversations.create()` / `openai_client.responses.create()` — handling the
function-call loop yourself.

**Estimated duration:** 45 minutes

**Prerequisites:** Modules 2–4 complete — the **Invoice Discrepancy Assistant** Prompt
Agent exists, grounds policy answers with File Search (Module 3), and can analyze
`discrepancy-trends.csv` with Code Interpreter (Module 4). You'll need your Foundry project
endpoint (noted in Module 1, Step 2) and a local clone of this lab repository.

## Why this module?

- **Where we are:** The assistant can retrieve company policy and analyze historical trends.
- **What's missing:** Neither capability gives it a tool to retrieve an actual purchase order.
  It cannot reliably compare an invoice with PO-10234 without that PO's recorded values.
- **What we'll add:** A custom PO lookup function and a Python conversation client. The model
  requests the tool; our client executes it and returns the result. We also learn to update the
  existing Prompt Agent's configuration from code while retaining its built-in tools.
- **Where it runs:** The Prompt Agent runs in Foundry. Your client and lookup function run on
  your computer and read the local sample CSV. **Azure Functions is not used.** Registering a
  `FunctionTool` describes the callable tool; it does not upload or host its Python implementation.
- **By the end:** Asking whether PO-10234 was invoiced within tolerance should combine
  retrieved PO values with company policy, rather than a guessed price or a refusal.

> **Why code now?** Modules 2–4 configured the agent entirely through the portal. Starting here,
> you'll extend the *same* Prompt Agent by creating new versions of it from code — this module
> adds a custom function tool; Module 6 adds an MCP tool the same way. This is still the one
> Prompt Agent from Module 2, just gaining capabilities over successive versions.
> It is **not** the code-first Agent Framework system you'll build in Module 8 — don't conflate
> the two.

## Steps

### 1. Set up your Python environment

1. Open a terminal in the working folder you created in Module 0:

   ```powershell
   Set-Location "$HOME\foundry-practice"
   ```

2. Create and activate a virtual environment:

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

3. Install the packages this module needs:

   ```powershell
   pip install azure-ai-projects azure-identity openai python-dotenv
   ```

4. Confirm you're signed in with an identity that has access to your Foundry project (the SDK
   uses `DefaultAzureCredential`, which picks up your `az login` session):

   ```powershell
   az account show --output table
   ```

### 2. Configure environment variables

1. In your working folder, create a file named `.env`:

   ```text
   PROJECT_ENDPOINT=https://<your-project-name>.<region>.ai.azure.com/api/projects/<project-name>
   MODEL_DEPLOYMENT=invoice-assistant-chat
   LAB_REPO_ROOT=C:\path\to\your\local\clone
   ```

   - `PROJECT_ENDPOINT` is the project endpoint you noted in Module 1, Step 2.
   - `MODEL_DEPLOYMENT` is the deployment name from Module 1.
   - `LAB_REPO_ROOT` is the path to the root of your local clone of this repository - the folder
     that directly contains a `data/` subfolder with `po-records.csv`, not a nested subfolder path.
     Your code will read sample data from there instead of duplicating it into your working folder.

2. Do not commit `.env` to source control — it's a good habit to add it to a `.gitignore` in your
   working folder even for a lab.

3. Before continuing, verify the path in PowerShell. PowerShell does not load `.env` automatically;
   set the session variable to the same value you entered in `.env` (replace the placeholder):

   ```powershell
   $env:LAB_REPO_ROOT = "C:\path\to\your\local\clone"
   ```

   Then run this one-line check:

   ```powershell
   Test-Path "$env:LAB_REPO_ROOT\data\po-records.csv"
   ```

   It should return `True`. If it returns `False`, `LAB_REPO_ROOT` is pointing at the wrong folder.
   Correct the value in both `.env` and the PowerShell session before continuing.

### 3. Review the PO records the lookup tool will read

1. Open [`data/po-records.csv`](/data/po-records.csv) in VS Code.
2. Note the columns: `po_number`, `vendor_name`, `item_description`, `unit_price`, `quantity`,
   `order_date`, `status` — and that `PO-10234` (the example you tried unsuccessfully in Module 2)
   is in there, so you can confirm the tool actually works this time.
3. Note that PO numbers have intentional gaps (for example, there's no `PO-10243`) — asking about
   a nearby, nonexistent PO number is a realistic way to trigger the missing-PO path later.

### 4. Write the PO lookup function

Create a file named `functions.py` in your working folder:

```python
"""Custom function tools for the Invoice Discrepancy Assistant.

One plain Python function per tool, each returning a JSON string the model can read
directly as a function-call output — mirroring the style of the mslearn-ai-agents
custom-tools exercise.
"""

import csv
import json
import os
from pathlib import Path

DATA_DIR = Path(os.environ["LAB_REPO_ROOT"]) / "data"
PO_RECORDS_FILE = DATA_DIR / "po-records.csv"


def lookup_po_by_number(po_number: str) -> str:
    """Look up a purchase order record by its PO number.

    Args:
        po_number: The PO number to look up, e.g. "PO-10234".

    Returns:
        A JSON string with the matching PO record (vendor, item, unit price, quantity,
        order date, and status), or a "not found" result if no record matches.
    """
    normalized = po_number.strip().upper()

    with PO_RECORDS_FILE.open(newline="", encoding="utf-8") as csv_file:
        reader = csv.DictReader(csv_file)
        for row in reader:
            if row["po_number"].strip().upper() == normalized:
                return json.dumps(
                    {
                        "found": True,
                        "po_number": row["po_number"],
                        "vendor_name": row["vendor_name"],
                        "item_description": row["item_description"],
                        "unit_price": float(row["unit_price"]),
                        "quantity": int(row["quantity"]),
                        "order_date": row["order_date"],
                        "status": row["status"],
                    }
                )

    return json.dumps(
        {
            "found": False,
            "po_number": po_number,
            "message": (
                f"No purchase order found matching '{po_number}'. Treat this as a "
                "missing-PO case per policy Section 6 rather than guessing a result."
            ),
        }
    )
```

> **Why return a "not found" JSON object instead of raising an exception?** The model needs a
> result it can reason about and relay to the analyst (per policy Section 6, a missing PO is a
> normal, well-defined case — not a tool failure). Raising an exception here would surface as a
> generic tool error instead of a meaningful "this PO doesn't exist" answer.

### 5. Define the `FunctionTool` and update the agent's instructions

Create a file named `client.py` in your working folder. Start with the imports, environment
setup, the `FunctionTool` definition, and the updated agent instructions:

```python
import json
import os

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    CodeInterpreterContainerAuto,
    CodeInterpreterTool,
    FileSearchTool,
    FunctionTool,
    PromptAgentDefinition,
)
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

from functions import lookup_po_by_number

load_dotenv()

project_endpoint = os.environ["PROJECT_ENDPOINT"]
model_deployment = os.environ["MODEL_DEPLOYMENT"]

AGENT_INSTRUCTIONS = """
You are the Invoice Discrepancy Assistant for Contoso Finance's accounts payable team.

Your role is to help AP analysts triage invoice discrepancies, specifically:
- Price mismatches (invoiced unit price differs from the purchase order)
- Quantity mismatches (invoiced quantity differs from the purchase order)
- Duplicate invoices
- Missing purchase orders

Guidelines:
- Always ground discrepancy-handling guidance (thresholds, escalation steps, approval rules) in
  Contoso's official policy — never invent a threshold, rule, or policy detail.
- You can analyze historical discrepancy trend data (dates, discrepancy types, vendors, amounts,
  resolution outcomes) using the attached code interpreter tool when asked a trend or
  aggregation question — run code against the data rather than estimating an answer from memory.
- Never invent or guess purchase order data (PO number, vendor, price, quantity, status). Only
  state PO details you can retrieve from an authoritative source.
- If you don't have enough information to answer confidently, say so and explain what
  information is missing, rather than guessing.
- Keep responses concise and action-oriented for a busy AP analyst.
- When you cite a policy rule, make it clear you're citing company policy.

You have access to Contoso's official invoice/PO discrepancy policy through file search — use it
to ground every threshold, escalation rule, or approval-limit statement you make. Cite the
relevant policy section when you reference a specific rule.

You now have a lookup_po_by_number tool — use it any time you need an authoritative PO's vendor,
item, unit price, quantity, order date, or status, and always use the retrieved values (never a
guess) when comparing against an invoice. If the tool reports no matching PO, treat that as a
missing-PO case per Section 6, not as an error to apologize for.
"""

po_lookup_tool = FunctionTool(
    name="lookup_po_by_number",
    description="Look up a purchase order record by its PO number.",
    parameters={
        "type": "object",
        "properties": {
            "po_number": {
                "type": "string",
                "description": "The PO number to look up, e.g. 'PO-10234'",
            },
        },
        "required": ["po_number"],
        "additionalProperties": False,
    },
    strict=True,
)
```

This is the same instructions text from Modules 2–4, with the final paragraph (which previously
said the agent didn't yet have a PO lookup tool) replaced with the paragraph granting it.

`create_version` replaces the entire tool list, so include File Search and Code Interpreter
alongside the PO lookup tool.

1. Module 3 created the vector store in the portal without capturing its ID. Run this standalone
   snippet from your working folder to list vector stores and find the Module 3 store by name:

   ```python
   import os

   from azure.ai.projects import AIProjectClient
   from azure.identity import DefaultAzureCredential
   from dotenv import load_dotenv

   load_dotenv()

   with (
       DefaultAzureCredential() as credential,
       AIProjectClient(
           endpoint=os.environ["PROJECT_ENDPOINT"], credential=credential
       ) as project_client,
       project_client.get_openai_client() as openai_client,
   ):
       for vector_store in openai_client.vector_stores.list():
           print(vector_store.name, vector_store.id)
   ```

   Alternatively, open that vector store's page in the portal and copy its ID.

2. Add the matching ID to your `.env` file:

   ```text
   FILE_SEARCH_VECTOR_STORE_ID=<your-Module-3-vector-store-ID>
   ```

3. Append these tool definitions to `client.py`, below `po_lookup_tool`:

   ```python
   file_search_tool = FileSearchTool(
       vector_store_ids=[os.environ["FILE_SEARCH_VECTOR_STORE_ID"]]
   )
   code_interpreter_tool = CodeInterpreterTool(container=CodeInterpreterContainerAuto())
   ```

### 6. Create a new agent version with the PO lookup tool

Append the following to `client.py`. The `tools=[...]` list includes all three tools:

```python
with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=project_endpoint, credential=credential) as project_client,
    project_client.get_openai_client() as openai_client,
):
    agent = project_client.agents.create_version(
        agent_name="Invoice Discrepancy Assistant",
        definition=PromptAgentDefinition(
            model=model_deployment,
            instructions=AGENT_INSTRUCTIONS,
            tools=[
                po_lookup_tool,
                file_search_tool,
                code_interpreter_tool,
            ],
        ),
    )
    print(f"Updated agent '{agent.name}' with the PO lookup tool.")
```

> **Why `agent_name="Invoice Discrepancy Assistant"` and not a new name?** Using the same name as
> the agent you created in Module 2 creates a new **version** of that agent rather than a
> separate agent — this is intentional. Every run of this script creates another version (which
> is harmless for a lab), but keep in mind that in a real project you'd typically only do this
> when you actually mean to change the agent's configuration.

### 7. Build the conversation client

Append the interactive client loop to the same `client.py`, still inside the `with` block from
Step 6:

```python
    conversation = openai_client.conversations.create()
    print("Invoice Discrepancy Assistant ready. Type 'exit' to quit.\n")

    while True:
        user_input = input("YOU: ").strip()
        if user_input.lower() in {"exit", "quit"}:
            break

        openai_client.conversations.items.create(
            conversation_id=conversation.id,
            items=[{"type": "message", "role": "user", "content": user_input}],
        )

        input_list: ResponseInputParam = []
        response = openai_client.responses.create(
            conversation=conversation.id,
            input=input_list,
            extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
        )

        while any(item.type == "function_call" for item in response.output):
            input_list = []
            for item in response.output:
                if item.type != "function_call":
                    continue
                if item.name != "lookup_po_by_number":
                    raise ValueError(f"Unknown function tool: {item.name}")

                result = lookup_po_by_number(**json.loads(item.arguments))
                input_list.append(
                    FunctionCallOutput(
                        type="function_call_output",
                        call_id=item.call_id,
                        output=result,
                    )
                )

            response = openai_client.responses.create(
                conversation=conversation.id,
                input=input_list,
                extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
            )

        print(f"AGENT: {response.output_text}\n")
```

> **`conversations`/`responses`, not threads/runs:** this client uses the
> `openai_client.conversations.create()` / `openai_client.responses.create()` API surface — not
> the older Assistants-style threads/runs API. The first `responses.create()` call can return
> zero or more `function_call` items in `response.output`; you execute each one locally, append a
> matching `FunctionCallOutput` to `input_list`, and call `responses.create()` again. Repeat until
> the latest response has no function calls, resetting `input_list` each round so outputs aren't
> submitted twice. Module 6 extends this exact loop to also handle local MCP tool calls, so keep
> this structure intact.

### 8. Run the client

1. From your activated virtual environment, run:

   ```powershell
   python client.py
   ```

2. Confirm you see `Updated agent '...' with the PO lookup tool.` followed by the `YOU:` prompt.
3. Try the PO lookup that failed back in Module 2:

   ```text
   Look up PO number PO-10234 for me.
   ```

   Confirm the agent now returns real data — vendor **Litware Manufacturing**, item
   **CNC-machined aluminum brackets**, unit price **$14.80**, quantity **1000** — instead of
   saying it can't retrieve it.

4. Try a PO number that doesn't exist:

   ```text
   Look up PO number PO-10243 for me.
   ```

   Confirm the agent reports it couldn't find that PO and frames it as a missing-PO case rather
   than apologizing or inventing data.

5. Try a question that should combine the new tool with the grounding/analysis from earlier
   modules:

   ```text
   PO-10234 was invoiced at $16.50 per unit for all 1000 units. Is that within our price
   mismatch tolerance, and what should I do?
   ```

   Confirm the agent calls `lookup_po_by_number`, compares $16.50 against the retrieved $14.80,
   and cites the correct tolerance tier from policy Section 3.1 (this variance is about 11.5%,
   well past the $250/5% escalate-to-Procurement threshold on 1000 units).

6. Type `exit` to end the session.

## ✅ Checkpoint / Validate

- [ ] `python client.py` runs without import or authentication errors and prints a confirmation
  that the agent version was updated.
- [ ] Asking to look up **PO-10234** returns the real record from `po-records.csv` (vendor,
  price, quantity), not a refusal.
- [ ] Asking to look up a nonexistent PO number (for example **PO-10243**) returns a clear
  "not found" response framed as a missing-PO case.
- [ ] A combined price-comparison question correctly calls the PO lookup tool **and** cites the
  right policy tolerance tier.
- [ ] You re-tested a Module 3 policy question and a Module 4 trend question against the new
  agent version, and both still work — confirming you didn't drop File Search or Code
  Interpreter configured in Step 5.

## 🚀 Try it yourself

Add a second function tool, `list_pos_by_vendor(vendor_name: str) -> str`, that returns every PO
record for a given vendor as a JSON array (reuse the CSV-reading pattern from
`lookup_po_by_number`). Add it to both the `tools=[...]` list in `client.py` and the
`response.output` handling loop, then ask:

```text
Show me every open PO we have with Fabrikam Industrial Supply.
```

Confirm the agent calls your new tool and lists only that vendor's records.

## ⚠️ Troubleshooting

- **`KeyError: 'PROJECT_ENDPOINT'` or similar on startup:** Confirm `.env` is in the same folder
  you're running `python client.py` from, and that `load_dotenv()` runs before you read
  `os.environ[...]`.
- **`FileNotFoundError` reading `po-records.csv`:** Confirm `LAB_REPO_ROOT` in `.env` points at
  the root of your local clone of this repository (the folder directly containing `data/po-records.csv`), not
  your separate working folder from Module 0.
- **Agent still refuses to look up a PO after Step 8:** Confirm `create_version` actually
  succeeded (check the printed confirmation message, and check **Agents** in the portal for a new
  version) and that `po_lookup_tool` is genuinely present in the `tools=[...]` list you ran, not
  just in an earlier draft of the file.
- **Module 3/4 questions stop working after this module:** Confirm your `tools=[...]` list includes
  `po_lookup_tool`, `file_search_tool`, and `code_interpreter_tool` as shown in Step 6, and that
  `FILE_SEARCH_VECTOR_STORE_ID` matches the Module 3 store. Restore any missing definitions and
  re-run `create_version`.
- **`response.output` has no `function_call` items even when you ask for a PO lookup:** Double
  check the `FunctionTool`'s `parameters` schema matches exactly what you call in Python
  (`po_number`), and that `strict=True` isn't rejecting a malformed schema — a schema/behavior
  mismatch here is a common cause of the model silently answering from its own text instead of
  calling the tool.
- **Authentication errors from `DefaultAzureCredential`:** Confirm `az login` is still valid
  (`az account show`) and that your identity has a role with access to the Foundry project (for
  example **Azure AI User** or an equivalent custom role) — an expired token or missing role
  assignment both surface as generic auth failures here.

