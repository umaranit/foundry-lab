# Module 6: MCP Integration (Code)

**Objective:** Build a local MCP server using FastMCP with an `@mcp.tool()`-decorated
escalation-contact lookup tool grounded in the invoice/PO policy's Section 8 table, connect to it
from the Module 5 (Custom Function Tools) client using the official `mcp` package's
`stdio_client` / `ClientSession`, wrap its tool as a `FunctionTool` on a new agent version, and
extend the conversation loop to call it.

> **Note:** This module adds a **local, stdio-based** MCP server — the kind you write and run
> yourself as a subprocess. There is no `MCPTool` class and no approval step involved anywhere in
> this module; those only apply to **remote** MCP servers accessed over HTTP, as outlined in
> [Module 7 (Foundry IQ Knowledge Base)](07-foundry-iq-knowledge-base.md). If you've seen the
> `MCPTool`/approval-request pattern elsewhere (for example in Foundry documentation about hosted
> remote MCP tools), set it aside for this module — it does not apply here. See Step 6 below for
> exactly what does apply instead.

**Estimated duration:** 45 minutes

**Prerequisites:** Module 5 (Custom Function Tools) complete — you have a working `client.py` and
`functions.py` in your Module 0 (Prerequisites) working folder, the **Invoice Discrepancy
Assistant** agent has a version with the `lookup_po_by_number` `FunctionTool` (plus File Search
and Code Interpreter), and your virtual environment is activated.

## Why this module?

- **Where we are:** Module 5 (Custom Function Tools) added a PO lookup implemented directly in our
  Python client.
- **What's missing:** We have not learned how to expose a tool through a separate server and
  a standard protocol. Policy retrieval can already answer contact questions, but it is not a
  dedicated structured contact lookup.
- **What we'll add:** A FastMCP server exposing escalation contacts and SLAs from policy
  Section 8, plus an async client connection that forwards tool requests to that server.
- **Where it runs:** The client and MCP server both run on your computer, communicating over
  stdin/stdout. Foundry sees a `FunctionTool`; your client bridges it to MCP. This is neither
  Azure Functions nor a remote HTTP MCP endpoint, and it has no MCP approval-request loop.
- **By the end:** Asking who handles a particular discrepancy and within what SLA should
  execute the MCP lookup and return the policy table's values, alongside the existing PO tool.

## Steps

### 1. Install the MCP packages

1. From the virtual environment you activated for Module 5 (Custom Function Tools), install
   `fastmcp` (which you'll use to write the server) and `mcp` (the official Model Context Protocol
   Python SDK, which you'll use to connect to it as a client):

   ```powershell
   pip install fastmcp mcp
   ```

### 2. Review the Section 8 escalation-contact table

1. Open [`data/invoice-po-policy.md`](../data/invoice-po-policy.md) and find **Section 8,
   "Escalation contact summary."** It's a small Markdown table with three columns: discrepancy
   type (for example `Price mismatch (Tier 2)` or `Missing PO (> $5,000)`), first escalation
   contact, and SLA.
2. Note the exact discrepancy-type strings — your tool will match against them, so the model
   needs to pass through a value from this table (or close to it) for a lookup to succeed.

### 3. Build the local MCP server

Create a file named `server.py` in your working folder:

```python
"""Local MCP server exposing an escalation-contact lookup tool.

A FastMCP server that reads directly from policy Section 8 so the tool's data can never drift
from the authoritative document — mirroring the "read the source file at call time" approach
functions.py used for po-records.csv in Module 5 (Custom Function Tools).
"""

import os
from pathlib import Path

from dotenv import load_dotenv
from fastmcp import FastMCP

# Load .env so the server also works when started directly (Step 4) in a fresh terminal.
load_dotenv()

POLICY_FILE = Path(os.environ["LAB_REPO_ROOT"]) / "data" / "invoice-po-policy.md"

mcp = FastMCP(name="EscalationContacts")


def _load_section_8_table() -> list[dict[str, str]]:
    """Parse the Markdown table in policy Section 8 into a list of row dicts."""
    text = POLICY_FILE.read_text(encoding="utf-8")
    section = text.split("## 8. Escalation contact summary")[1].split("## 9.")[0]
    table_lines = [line for line in section.splitlines() if line.strip().startswith("|")]

    # table_lines[0] is the header row, table_lines[1] is the "---|---|---" separator.
    records = []
    for line in table_lines[2:]:
        cells = [cell.strip() for cell in line.strip().strip("|").split("|")]
        records.append(
            {
                "discrepancy_type": cells[0],
                "first_escalation_contact": cells[1],
                "sla": cells[2],
            }
        )
    return records


@mcp.tool()
def get_escalation_contact(discrepancy_type: str) -> dict:
    """Look up the first escalation contact and SLA for a discrepancy type.

    Args:
        discrepancy_type: The discrepancy type exactly as it appears in policy Section 8,
            e.g. "Price mismatch (Tier 2)" or "Missing PO (> $5,000)".

    Returns:
        A dict with the matching contact and SLA, or a "not found" result listing the valid
        discrepancy types if there's no exact match.
    """
    normalized = discrepancy_type.strip().lower()

    for record in _load_section_8_table():
        if record["discrepancy_type"].strip().lower() == normalized:
            return {"found": True, **record}

    return {
        "found": False,
        "discrepancy_type": discrepancy_type,
        "message": "No exact match in policy Section 8.",
        "valid_discrepancy_types": [r["discrepancy_type"] for r in _load_section_8_table()],
    }


if __name__ == "__main__":
    mcp.run(show_banner=False)
```

> **Why `-> dict` instead of `-> str` like `lookup_po_by_number` in
> Module 5 (Custom Function Tools)?** FastMCP inspects the return type annotation to build the
> tool's output schema. A `dict` return is sent back as structured data *and* as a JSON text block
> the model can read directly — you don't need to call `json.dumps` yourself the way `functions.py`
> did in Module 5 (Custom Function Tools).

### 4. Confirm the server starts cleanly

1. Run the server directly once, just to confirm there are no import or path errors:

   ```powershell
   python server.py
   ```

2. You should see a single startup line similar to:

   ```text
   INFO     Starting MCP server 'EscalationContacts' with transport 'stdio'
   ```

   Nothing else will print, and you don't need to wait — `mcp.run()` puts the process into a loop
   waiting for a client to talk to it over stdin/stdout (this is the "stdio transport"). Seeing
   this line means the server started successfully. Press `Ctrl+C` to stop it.

   > **Note:** When you press `Ctrl+C`, Python may print a long traceback (mentioning `anyio`,
   > `asyncio`, `KeyboardInterrupt`, or `CancelledError`). That's normal shutdown output from
   > interrupting the server — not an error. Only a traceback that appears *before* the startup
   > line (for example, `KeyError: 'LAB_REPO_ROOT'`) indicates a real problem.
3. You won't normally launch `server.py` directly like this again — in Step 6, your client
   launches it automatically as a subprocess each time it runs.

### 5. Add the escalation-contact paragraph to the agent's instructions

Open `client.py` from Module 5 (Custom Function Tools) and add one paragraph to the end of
`AGENT_INSTRUCTIONS`, alongside the existing PO-lookup paragraph:

```text
You now also have a get_escalation_contact tool — use it any time an analyst needs to know who to
escalate a discrepancy to or what SLA applies, passing the discrepancy type exactly as classified
per policy (e.g. "Price mismatch (Tier 2)", "Missing PO (> $5,000)"). Always use the tool's
answer rather than recalling a contact or SLA from memory.
```

### 6. Connect to the MCP server and make the client async

This is the biggest structural change in this module, so read this before editing code.

> **Why does the client need to become async?** The `client.py` from
> Module 5 (Custom Function Tools) is a synchronous script — `AIProjectClient`, `openai_client`,
> and the `input()`-driven loop are all plain, blocking Python.
> The `mcp` package's `stdio_client` and `ClientSession`, however, are **async context managers**:
> they manage the MCP server subprocess and the JSON-RPC messages flowing to/from it over asyncio
> streams, and only work inside `async with` blocks and `await`ed calls. Rather than mixing sync
> and async code awkwardly, this module moves the whole client body into one `async def main()`
> function and runs it with `asyncio.run(main())`. The existing synchronous Azure SDK and
> `openai_client` calls still work fine *inside* an async function — you just don't `await` them.

Replace the imports at the top of `client.py` with:

```python
import asyncio
import json
import os
import sys

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
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

load_dotenv()

from functions import lookup_po_by_number
```

Keep `po_lookup_tool`, `file_search_tool`, and `code_interpreter_tool` at module level, unchanged
from Module 5 (Custom Function Tools), including `FILE_SEARCH_VECTOR_STORE_ID` and
`CODE_INTERPRETER_FILE_ID` in `.env`. Below them, add the MCP server launch parameters:

```python
server_params = StdioServerParameters(
    command=sys.executable,
    args=["server.py"],
    env={"LAB_REPO_ROOT": os.environ["LAB_REPO_ROOT"]},
)
```

> `sys.executable` uses the client process's Python interpreter. `env=` explicitly passes
> `LAB_REPO_ROOT` to the server because the full subprocess environment is not inherited by default.

In `client.py`, select the **entire existing `with` block**, not just its first few lines:

- Start at `with (` immediately before `DefaultAzureCredential()`.
- Include the agent creation, conversation setup, and entire `while True` loop inside the block.
- End at `print(f"AGENT: {response.output_text}\n")`, the final line of the conversation loop.

Replace that entire selection with the code below. Keep the configuration, tool definitions
(`po_lookup_tool`, `file_search_tool`, and `code_interpreter_tool`), and `server_params` above
the selection. The old `with` block is removed for now; Step 7 adds a new one inside `main()`,
and Step 8 adds the new conversation loop. Complete Steps 6–8 before running the client.

```python
async def main():
    async with stdio_client(server_params) as (stdio, write):
        async with ClientSession(stdio, write) as session:
            await session.initialize()
            tools_response = await session.list_tools()
            tools = tools_response.tools
            print("Connected to MCP server with tools:", [t.name for t in tools])

            def make_tool_func(tool_name):
                async def tool_func(**kwargs):
                    return await session.call_tool(tool_name, kwargs)

                tool_func.__name__ = tool_name
                return tool_func

            functions_dict = {t.name: make_tool_func(t.name) for t in tools}

            mcp_function_tools = [
                FunctionTool(
                    name=t.name,
                    description=t.description,
                    parameters=t.input_schema,
                    strict=False,
                )
                for t in tools
            ]
```

Each function uses its own MCP input schema. `strict=False` preserves that schema without
requiring it to satisfy the stricter FunctionTool schema constraints.

> **No `MCPTool`, no approval step.** `functions_dict` and `mcp_function_tools` are the *entire*
> MCP integration on the agent-definition side: each MCP tool becomes an ordinary `FunctionTool`
> the agent already knows how to call, exactly like `po_lookup_tool` from
> Module 5 (Custom Function Tools). There's no separate tool type to add and nothing to approve —
> approval requests are a **remote** MCP concept (the agent calling a server it doesn't control
> over HTTP) that doesn't apply when **you** are the one launching the server as a local subprocess.

### 7. Add code to create the new agent version

In `client.py`, **keep the code you added in Step 6**. Go to the end of that code block, after
the closing `]` of `mcp_function_tools`, and add the following `with` block on a new line.
Indent its `with (` line to the same level as `tools_response = ...` in Step 6 (12 spaces in
the code shown). This puts it inside `async with ClientSession(...)` but after the MCP
connection and tool list have been prepared.

This is a **new** `with` block for the Azure clients; it is not the old block from
Module 5 (Custom Function Tools) that you removed in Step 6. Do not remove or replace the
`async def main()`, either `async with` block, or the MCP tool setup above it.

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
                            *mcp_function_tools,
                        ],
                    ),
                )
                print(f"Updated agent '{agent.name}' to version {agent.version} with the escalation-contact MCP tool.")
```

This code creates the new agent version only when you run `client.py` in Step 9. `create_version`
replaces the entire tool list, so the list above retains all three tools from
Module 5 (Custom Function Tools) and adds the MCP tools. Re-test questions from
Module 3 (Knowledge Base / Grounding), Module 4 (Tools — Built-in), and
Module 5 (Custom Function Tools) after Step 9 to confirm nothing regressed.

### 8. Extend the conversation loop to call the MCP tool

Still inside the same `with` block, append the conversation loop — structurally the same as
Module 5 (Custom Function Tools), Step 7, but the function-call handling now dispatches to
either the local `lookup_po_by_number` or an MCP tool via `functions_dict`:

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

                            kwargs = json.loads(item.arguments)

                            if item.name == "lookup_po_by_number":
                                result = lookup_po_by_number(**kwargs)
                            elif item.name in functions_dict:
                                output = await functions_dict[item.name](**kwargs)
                                result = output.content[0].text
                            else:
                                raise ValueError(f"Unknown function tool: {item.name}")

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


if __name__ == "__main__":
    asyncio.run(main())
```

> **`await functions_dict[item.name](**kwargs)` and `output.content[0].text`:** calling an MCP
> tool through `ClientSession.call_tool` is async (hence the `await`), and it returns a
> `CallToolResult` whose `content` list holds the response as one or more content blocks. Because
> `get_escalation_contact` returns a `dict`, FastMCP serializes it into a JSON text block, so
> `output.content[0].text` is the JSON string to hand back as the `FunctionCallOutput`'s `output`
> — the same shape `lookup_po_by_number` already produces directly as a string.

### 9. Run the client

1. From your activated virtual environment, run:

   ```powershell
   python client.py
   ```

2. Confirm you see `Connected to MCP server with tools: ['get_escalation_contact']`, then a
   message like `Updated agent 'invoice-discrepancy-assistant' to version 6 with the
   escalation-contact MCP tool.`, followed by the `YOU:` prompt.
3. Before continuing to Module 7 (Foundry IQ Knowledge Base), open the `.env` file in the same
   working folder as `client.py`. Add this line, or update it if it already exists, replacing `6`
   with the version number printed in your terminal (do not include angle brackets):

   ```text
   EVAL_AGENT_VERSION=6
   ```

   Module 10 (Evaluation) and Module 11 (Monitoring & Tracing) use this value to target the
   Module 6 (MCP Integration) agent version.
4. **Verify in the Foundry portal:** Open [Foundry](https://ai.azure.com) and navigate to your
   project > **Agents** > `invoice-discrepancy-assistant` > **Versions**. Confirm that the
   version number printed in the terminal appears with `get_escalation_contact` alongside
   `lookup_po_by_number`, **File Search**, and **Code Interpreter**.
5. Ask an escalation question:

   ```text
   A price mismatch just got classified as Tier 2. Who should I escalate it to and what's the SLA?
   ```

   Confirm the agent calls `get_escalation_contact` and answers **AP Supervisor** within
   **2 business days**, matching policy Section 8.
6. Ask about a case with no internal escalation:

   ```text
   We have a missing PO on an $8,000 invoice. Who handles the escalation?
   ```

   Confirm the agent reports the invoice is rejected back to the vendor with no internal
   escalation (per the "Missing PO (> $5,000)" row), rather than inventing a contact.
7. Re-test a Module 5 (Custom Function Tools) question (for example, looking up `PO-10234`) to
   confirm `po_lookup_tool` still works on this new agent version.
8. Type `exit` to end the session.

## ✅ Checkpoint / Validate

- [ ] `python client.py` prints the connected-tools line, then the agent-version confirmation,
  without import or authentication errors.
- [ ] The Foundry portal's **Versions** tab shows a new **Invoice Discrepancy Assistant** version
  with `get_escalation_contact` listed alongside all three tools from
  Module 5 (Custom Function Tools).
- [ ] Asking an escalation-contact question returns the correct contact and SLA from policy
  Section 8, not a guessed answer.
- [ ] Asking about the "no internal escalation" case (missing PO over $5,000) is handled
  correctly, not as a tool error.
- [ ] A Module 5 (Custom Function Tools) PO-lookup question and a policy or trend question from
  Module 3 (Knowledge Base / Grounding) or Module 4 (Tools — Built-in) still work against this new
  agent version.

## 🚀 Try it yourself

Add a second `@mcp.tool()` to `server.py` — for example `get_sla_business_days(discrepancy_type:
str) -> dict`, returning just the numeric SLA business-day count (or `None` for the no-escalation
case) parsed from the same Section 8 table. You don't need to touch `client.py` at all: because
`mcp_function_tools` is built by iterating `tools_response.tools`, any new `@mcp.tool()` you add
to the server is picked up and exposed to the agent automatically the next time you run
`client.py`. Confirm the agent can call it with a question like:

```text
How many business days do we have to resolve a duplicate invoice escalation?
```

## ⚠️ Troubleshooting

- **`KeyError: 'LAB_REPO_ROOT'` when running `python server.py`:** The server couldn't find
  `LAB_REPO_ROOT`. Confirm `server.py` includes `load_dotenv()` (Step 3), that you run it from the
  working folder containing the `.env` file from Module 5 (Custom Function Tools), and that `.env`
  has a `LAB_REPO_ROOT=...` line.
- **Client hangs or errors immediately after "Connected to MCP server..." never prints:** The
  subprocess FastMCP server likely failed to start. Confirm `python server.py` runs cleanly on
  its own (Step 4), that `fastmcp` and `mcp` are installed in the client interpreter, and that
  `env=` passes `LAB_REPO_ROOT` to the server.
- **`AttributeError: 'Tool' object has no attribute 'inputSchema'`:** In `client.py`, change
  `parameters=t.inputSchema` to `parameters=t.input_schema` in the `mcp_function_tools`
  definition from Step 6, then rerun the client.
- **`TypeError: object ... can't be used in 'await' expression` or similar async errors:** This
  usually means a sync call got left inside an `await`, or an `async def` got called without
  `await`/`asyncio.run`. Compare your `client.py` against Step 6–8 closely — the whole client body
  must be inside `async def main()`, invoked only via `asyncio.run(main())` at the bottom of the
  file, not called directly.
- **`RuntimeError: This event loop is already running` or nested `asyncio.run` errors:** Make sure
  you only call `asyncio.run(main())` once, at module level under `if __name__ == "__main__":`,
  and that you haven't also wrapped the script in another async runner.
- **Escalation-contact answers look wrong or stale:** Confirm `LAB_REPO_ROOT` in `.env`, set in
  Module 5 (Custom Function Tools), still points at the root of your local clone of this repository
  (the folder directly containing `data/po-records.csv`), so `server.py` reads the same
  `invoice-po-policy.md` you reviewed in Step 2.
- **Agent answers from memory instead of calling `get_escalation_contact`:** Confirm Step 5's
  instructions paragraph was actually added, and that the discrepancy-type phrase in your question
  reasonably matches a Section 8 row — an unrecognized phrase may lead the model to answer
  generically instead of calling the tool.
- **Questions from Module 3 (Knowledge Base / Grounding), Module 4 (Tools — Built-in), or
  Module 5 (Custom Function Tools) stop working after this module:** Confirm your `tools=[...]`
  list includes `po_lookup_tool`, `file_search_tool`, `code_interpreter_tool`, and
  `*mcp_function_tools` as shown in Step 7, and that `CODE_INTERPRETER_FILE_ID` matches the
  Module 4 (Tools — Built-in) file. Restore any missing definitions from
  Module 5 (Custom Function Tools) and re-run `client.py`.
