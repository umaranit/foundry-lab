# Module 5: Custom Function Tools (Code)

**Objective:** Define a `FunctionTool` for "look up PO by number" (backed by `po-records.csv`),
create a new version of the **Invoice Discrepancy Assistant** Prompt Agent in code with
`project_client.agents.create_version`, and build a client that calls it using
`openai_client.conversations.create()` / `openai_client.responses.create()` — handling the
function-call loop yourself.

**Estimated duration:** 45 minutes

**Prerequisites:** An existing **Invoice Discrepancy Assistant** Prompt Agent with File Search
attached to `invoice-po-policy.md` and Code Interpreter attached to `discrepancy-trends.csv`.
You'll also need your Foundry project endpoint and a local copy of the lab's sample data.

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

> **Why code now?** The agent was configured through the portal. Starting here, you'll extend the
> *same* Prompt Agent by creating new versions of it from code — this module adds a custom
> function tool; Module 6 adds an MCP tool the same way. This is still the same Prompt Agent,
> just gaining capabilities over successive versions.
> It is **not** the code-first Agent Framework system you'll build in Module 8 — don't conflate
> the two.

### What are we simulating?

Imagine an Accounts Payable analyst receives an invoice for **PO-10234** and needs to check
whether its prices match the purchase order and company policy. In Module 4, the assistant learned
to analyze historical discrepancy trends, but it still could not retrieve the details of this
specific PO. This module adds that missing lookup.

The lab simulates a small business application:

1. **`client.py`** sends the analyst's question to the Foundry assistant.
2. The assistant requests the PO lookup, and `client.py` runs **`functions.py`** on your computer.
3. **`functions.py`** reads the PO from the local `data/po-records.csv` file and returns the
   matching record to the assistant.
4. The assistant compares the PO with the invoice and uses the existing policy and, when needed,
   trend-analysis tools to explain what the analyst should do.

The new agent version includes **File Search** and **Code Interpreter** again, alongside the PO
lookup, so it keeps the capabilities from Modules 3 and 4. The PO CSV stays on your computer;
only the result of a requested lookup is sent to Foundry. This is a lab example, not a live ERP
connection: in a real application, the lookup function could call a secured purchasing-system API.
The assistant recommends a next step; it does not hold payment or create an escalation case.

`functions.py` handles the data lookup; `client.py` handles the Foundry conversation and tool-call
loop. They are separate to make those responsibilities easier to understand and change.

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
PROJECT_ENDPOINT=https://<resource-name>.services.ai.azure.com/api/projects/<project-name>
   MODEL_DEPLOYMENT=invoice-assistant-chat
   LAB_REPO_ROOT=C:\path\to\your\local\clone
   ```

- `PROJECT_ENDPOINT` is the **Project endpoint** you copied from your project's Overview page
  in Module 1, Step 2. Paste the complete value exactly as shown; do not construct it from
  your project name or use the separate Azure OpenAI endpoint. In the example, `<resource-name>`
  is the Foundry resource name and `<project-name>` is the project name; they may differ.
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

1. Open [`data/po-records.csv`](../data/po-records.csv) in VS Code.
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

### 5. Define the tools and agent instructions

#### 5.1 Create the client

Create `client.py` in the same working folder as `functions.py` and `.env`. Paste the following
code into it and save the file. This defines the agent's instructions and the PO lookup tool:

```python
import json
import os

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    AutoCodeInterpreterToolParam,
    CodeInterpreterTool,
    FileSearchTool,
    FunctionTool,
    PromptAgentDefinition,
)
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

load_dotenv()

from functions import lookup_po_by_number

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

#### 5.2 Find the existing tools' resource IDs

To keep the agent's existing capabilities, you need two IDs:

| Capability | Resource whose ID you need |
|---|---|
| File Search | The **vector store** (index) containing `invoice-po-policy.md` |
| Code Interpreter | The uploaded **`discrepancy-trends.csv` file** |

Create `list_resources.py` in the same folder as `client.py`. Paste the following code and save it:

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
    for file in openai_client.files.list():
        print(file.filename, file.id)
```

In the VS Code terminal, make sure you're in the folder containing `list_resources.py`:

```powershell
Get-Location
Test-Path .\list_resources.py
```

`Test-Path` should return `True`. If it returns `False`, change to your working folder using
`Set-Location "C:\path\to\your\working-folder"` (use the **folder path**, not the filename).

With the virtual environment from Step 1 activated, run:

```powershell
python .\list_resources.py
```

If you opened a new terminal, activate the environment first using its actual path, for example
`& "C:\path\to\.venv\Scripts\Activate.ps1"`. Changing folders does not deactivate an already
active environment.

The output shows resource names followed by their IDs. For example (**illustrative IDs only**):

```text
policy-index              vs_example123
discrepancy-trends.csv    assistant-example456
invoice-po-policy.md      assistant-example789
```

Copy the ID beside your policy vector store's name and the ID beside `discrepancy-trends.csv`.
The ID beside `invoice-po-policy.md` is a file ID, **not** the vector store ID. Your names and
IDs will differ from the example. If multiple stores or files are listed, check the agent's
File Search and Code Interpreter configuration in the portal to identify the attached ones.

#### 5.3 Add the IDs to `.env`

Add these lines to the existing `.env` file, keeping its other settings. Replace the placeholders
with the IDs from your output, without the `<` and `>` characters, and save the file:

```text
FILE_SEARCH_VECTOR_STORE_ID=<policy-vector-store-id>
CODE_INTERPRETER_FILE_ID=<discrepancy-trends-file-id>
```

#### 5.4 Add the existing tools to `client.py`

Append these definitions below `po_lookup_tool` in `client.py`, then save it:

```python
file_search_tool = FileSearchTool(
    vector_store_ids=[os.environ["FILE_SEARCH_VECTOR_STORE_ID"]]
)
code_interpreter_tool = CodeInterpreterTool(
    container=AutoCodeInterpreterToolParam(
        file_ids=[os.environ["CODE_INTERPRETER_FILE_ID"]]
    )
)
```

### 6. Create a new agent version with the PO lookup tool

Now that `client.py` defines the PO lookup, File Search, and Code Interpreter tools, use
`project_client.agents.create_version` to create a new version of the existing agent.

The `tools` argument is the complete tool list for this version; tools attached to the portal
aren't carried over automatically. Include all three tools here to keep the existing policy
search and trend analysis capabilities while adding PO lookup.

Append the following to `client.py`:

```python
with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=project_endpoint, credential=credential) as project_client,
    project_client.get_openai_client() as openai_client,
):
    agent = project_client.agents.create_version(
        agent_name="invoice-discrepancy-assistant",
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

> **Why `agent_name="invoice-discrepancy-assistant"` and not a new name?** Using the same name as
> the agent you created in Module 2 creates a new **version** of that agent rather than a
> separate agent — this is intentional. Every run of this script creates another version (which
> is harmless for a lab), but keep in mind that in a real project you'd typically only do this
> when you actually mean to change the agent's configuration.

### 7. Build the conversation client

In `client.py`, go to the end of the code you added in Step 6. Immediately after the line that
prints the updated agent name, paste the code below. Keep the four spaces before `conversation`
and the other top-level lines in this block so they line up with the `agent =` and `print(...)`
lines in Step 6. This keeps the conversation loop inside the existing `with (...)` block. Do not
add another `with` statement.

The end of your file should look like this:

```python
    print(f"Updated agent '{agent.name}' with the PO lookup tool.")

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

1. Make sure the terminal is using the virtual environment from Step 1. Check which Python
   `python` refers to:

   ```powershell
   python -c "import sys; print(sys.executable)"
   ```

   The path should point to your `.venv\Scripts\python.exe`. If it does not, activate the
   environment using its actual path, for example:

   ```powershell
   & "C:\path\to\your\.venv\Scripts\Activate.ps1"
   ```

   Replace the example with the path to the `.venv` you created in Step 1. Then run:

   ```powershell
   python client.py
   ```

2. Confirm you see `Updated agent '...' with the PO lookup tool.` followed by the `YOU:` prompt.
3. In the Foundry portal, open your project > **Agents** >
   `invoice-discrepancy-assistant` > **Versions**. Confirm the new version lists
   `lookup_po_by_number`, **File Search**, and **Code Interpreter**. Confirm this is the same
   `invoice-discrepancy-assistant` agent created in Module 2 and that its version number has
   increased. If this run created a separate, differently named agent, stop and check that the
   agent name in `client.py` exactly matches the Module 2 name and that `PROJECT_ENDPOINT` points
   to the same Foundry project.
4. Try the PO lookup that failed back in Module 2:

   ```text
   Look up PO number PO-10234 for me.
   ```

   Confirm the agent now returns real data — vendor **Litware Manufacturing**, item
   **CNC-machined aluminum brackets**, unit price **$14.80**, quantity **1000** — instead of
   saying it can't retrieve it.

5. Try a PO number that doesn't exist:

   ```text
   Look up PO number PO-10243 for me.
   ```

   Confirm the agent reports it couldn't find that PO and frames it as a missing-PO case rather
   than apologizing or inventing data.

6. Try a question that should combine the new tool with the grounding/analysis from earlier
   modules:

   ```text
   PO-10234 was invoiced at $16.50 per unit for all 1000 units. Is that within our price
   mismatch tolerance, and what should I do?
   ```

   Confirm the agent calls `lookup_po_by_number`, compares $16.50 against the retrieved $14.80,
   and cites the correct tolerance tier from policy Section 3.1 (this variance is about 11.5%,
   well past the $250/5% escalate-to-Procurement threshold on 1000 units).

7. Type `exit` to end the session.

## ✅ Checkpoint / Validate

- [ ] `python client.py` runs without import or authentication errors and prints a confirmation
  that the agent version was updated.
- [ ] The Foundry portal's **Versions** tab shows a new `invoice-discrepancy-assistant` version
  with `lookup_po_by_number`, **File Search**, and **Code Interpreter** listed.
- [ ] The new version belongs to the same `invoice-discrepancy-assistant` agent created in
  Module 2, not a separately named agent.
- [ ] Asking to look up **PO-10234** returns the real record from `po-records.csv` (vendor,
  price, quantity), not a refusal.
- [ ] Asking to look up a nonexistent PO number (for example **PO-10243**) returns a clear
  "not found" response framed as a missing-PO case.
- [ ] A combined price-comparison question correctly calls the PO lookup tool **and** cites the
  right policy tolerance tier.
- [ ] You re-tested a policy question (for example, "What's the automatic price-mismatch
  approval threshold?") and a trend question (for example, "Which discrepancy type increased
  most from Q1 to Q3 2026?") against the new agent version. Both still work, confirming File
  Search and Code Interpreter are available.

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
  the folder directly containing the `data` folder and `data/po-records.csv`.
- **Agent still refuses to look up a PO after Step 8:** Confirm `create_version` actually
  succeeded (check the printed confirmation message, and check **Agents** in the portal for a new
  version) and that `po_lookup_tool` is genuinely present in the `tools=[...]` list you ran, not
  just in an earlier draft of the file.
- **Policy or trend questions stop working:** Confirm your `tools=[...]` list includes
  `po_lookup_tool`, `file_search_tool`, and `code_interpreter_tool` as shown in Step 6. Check that
  `FILE_SEARCH_VECTOR_STORE_ID` identifies the policy vector store and
  `CODE_INTERPRETER_FILE_ID` identifies the uploaded `discrepancy-trends.csv`. Restore any
  missing definitions and re-run `create_version`.
- **`response.output` has no `function_call` items even when you ask for a PO lookup:** Double
  check the `FunctionTool`'s `parameters` schema matches exactly what you call in Python
  (`po_number`), and that `strict=True` isn't rejecting a malformed schema — a schema/behavior
  mismatch here is a common cause of the model silently answering from its own text instead of
  calling the tool.
- **Authentication errors from `DefaultAzureCredential`:** Confirm `az login` is still valid
  (`az account show`) and that your identity has a role with access to the Foundry project (for
  example **Azure AI User** or an equivalent custom role) — an expired token or missing role
  assignment both surface as generic auth failures here.

