# Module 6: MCP Integration (Code)

**Objective:** Build a local MCP server using FastMCP with an `@mcp.tool()`-decorated
escalation-contact lookup tool grounded in the invoice/PO policy's Section 8 table, connect to it
from the Module 5 client using the official `mcp` package's `stdio_client` / `ClientSession`, wrap
its tool as a `FunctionTool` on a new agent version, and extend the conversation loop to call it.

> **Note:** This module adds a **local, stdio-based** MCP server — the kind you write and run
> yourself as a subprocess. There is no `MCPTool` class and no approval step involved anywhere in
> this module; those only apply to **remote** MCP servers accessed over HTTP, which this lab does
> not cover. If you've seen the `MCPTool`/approval-request pattern elsewhere (for example in
> Foundry documentation about hosted remote MCP tools), set it aside for this module — it does
> not apply here. See Step 6 below for exactly what does apply instead.

**Estimated duration:** 45 minutes

**Prerequisites:** Module 5 complete — you have a working `client.py` and `functions.py` in your
Module 0 working folder, the **Invoice Discrepancy Assistant** agent has a version with the
`lookup_po_by_number` `FunctionTool` (plus File Search and Code Interpreter), and your virtual
environment is activated.

## Steps

### 1. Install the MCP packages

1. From your activated virtual environment (the same one from Module 5), install `fastmcp` (which
   you'll use to write the server) and `mcp` (the official Model Context Protocol Python SDK,
   which you'll use to connect to it as a client):

   ```powershell
   pip install fastmcp mcp
   ```

### 2. Review the Section 8 escalation-contact table

1. Open [`data/invoice-po-policy.md`](/lab/data/invoice-po-policy.md) and find **Section 8,
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
functions.py used for po-records.csv in Module 5.
"""

import os
from pathlib import Path

from fastmcp import FastMCP

POLICY_FILE = Path(os.environ["LAB_REPO_ROOT"]) / "lab" / "data" / "invoice-po-policy.md"

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

> **Why `-> dict` instead of `-> str` like Module 5's `lookup_po_by_number`?** FastMCP inspects
> the return type annotation to build the tool's output schema. A `dict` return is sent back as
> structured data *and* as a JSON text block the model can read directly — you don't need to call
> `json.dumps` yourself the way `functions.py` did in Module 5.

### 4. Confirm the server starts cleanly

1. Run the server directly once, just to confirm there are no import or path errors:

   ```powershell
   python server.py
   ```

2. It won't print anything further — `mcp.run()` puts the process into a loop waiting for a
   client to talk to it over stdin/stdout (this is the "stdio transport"). That's expected; press
   `Ctrl+C` to stop it.
3. You won't normally launch `server.py` directly like this again — in Step 6, your client
   launches it automatically as a subprocess each time it runs.

### 5. Add the escalation-contact paragraph to the agent's instructions

Open `client.py` (from Module 5) and add one paragraph to the end of `AGENT_INSTRUCTIONS`,
alongside the existing PO-lookup paragraph from Module 5:

```text
You now also have a get_escalation_contact tool — use it any time an analyst needs to know who to
escalate a discrepancy to or what SLA applies, passing the discrepancy type exactly as classified
per policy (e.g. "Price mismatch (Tier 2)", "Missing PO (> $5,000)"). Always use the tool's
answer rather than recalling a contact or SLA from memory.
```

### 6. Connect to the MCP server and make the client async

This is the biggest structural change in this module, so read this before editing code.

> **Why does the client need to become async?** Module 5's `client.py` is a synchronous script —
> `AIProjectClient`, `openai_client`, and the `input()`-driven loop are all plain, blocking Python.
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

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FunctionTool, PromptAgentDefinition
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

from functions import lookup_po_by_number
```

Below `po_lookup_tool` (still at module level, unchanged from Module 5), add the MCP server
launch parameters:

```python
server_params = StdioServerParameters(command="python", args=["server.py"])
```

> **⚠️ `command="python"` must resolve to your activated virtual environment's Python.**
> `stdio_client` launches the server as a subprocess using whatever `python` resolves to on your
> `PATH` at that moment — it does **not** inherit your IDE's selected interpreter automatically.
> Launch `client.py` from the same activated-venv terminal you used in Step 4, so `fastmcp` is
> importable in the subprocess too.

Now replace the `with (...)` block from Module 5 (Steps 6–7 of that module) with an async version.
Everything from here through the end of the conversation loop now lives inside `async def main()`:

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
                    parameters={
                        "type": "object",
                        "properties": {
                            "discrepancy_type": {
                                "type": "string",
                                "description": (
                                    "The discrepancy type exactly as it appears in policy "
                                    "Section 8, e.g. 'Price mismatch (Tier 2)'."
                                ),
                            },
                        },
                        "required": ["discrepancy_type"],
                        "additionalProperties": False,
                    },
                    strict=True,
                )
                for t in tools
            ]
```

> **No `MCPTool`, no approval step.** `functions_dict` and `mcp_function_tools` are the *entire*
> MCP integration on the agent-definition side: each MCP tool becomes an ordinary `FunctionTool`
> the agent already knows how to call, exactly like `po_lookup_tool` from Module 5. There's no
> separate tool type to add and nothing to approve — approval requests are a **remote** MCP
> concept (the agent calling a server it doesn't control over HTTP) that doesn't apply when
> **you** are the one launching the server as a local subprocess.

### 7. Create the new agent version

Still inside `async def main()`, nested one level deeper (inside the `ClientSession` block), add
the `with` block that creates the new agent version — the same pattern as Module 5, Step 6:

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
                            *mcp_function_tools,
                            # ⚠️ Add your existing File Search and Code Interpreter tool
                            # definitions here too — see Module 5's "Verify before running"
                            # callout. create_version replaces the entire tools=[...] list every
                            # time, so the same rule from Module 5 applies again here: leaving
                            # them out silently drops File Search and Code Interpreter for this
                            # new version.
                        ],
                    ),
                )
                print(f"Updated agent '{agent.name}' with the escalation-contact MCP tool.")
```

> ## ⚠️ Verify before running
>
> This is the same warning as Module 5, Step 5 — it applies again here because `create_version`
> still replaces the entire tool list. If your `tools=[...]` list here is only
> `[po_lookup_tool, *mcp_function_tools]`, this new version drops File Search and Code Interpreter
> and (if you didn't carry it over from Module 5) could also drop `po_lookup_tool` support for
> Module 5's scenarios. Re-test Modules 3–5's questions after Step 9 below to confirm nothing
> regressed.

### 8. Extend the conversation loop to call the MCP tool

Still inside the same `with` block, append the conversation loop — structurally the same as
Module 5, Step 7, but the function-call handling now dispatches to either the local
`lookup_po_by_number` or an MCP tool via `functions_dict`:

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
                            continue

                        input_list.append(
                            FunctionCallOutput(
                                type="function_call_output",
                                call_id=item.call_id,
                                output=result,
                            )
                        )

                    if input_list:
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

2. Confirm you see `Connected to MCP server with tools: ['get_escalation_contact']`, then
   `Updated agent '...' with the escalation-contact MCP tool.`, followed by the `YOU:` prompt.
3. Ask an escalation question:

   ```text
   A price mismatch just got classified as Tier 2. Who should I escalate it to and what's the SLA?
   ```

   Confirm the agent calls `get_escalation_contact` and answers **AP Supervisor** within
   **2 business days**, matching policy Section 8.
4. Ask about a case with no internal escalation:

   ```text
   We have a missing PO on an $8,000 invoice. Who handles the escalation?
   ```

   Confirm the agent reports the invoice is rejected back to the vendor with no internal
   escalation (per the "Missing PO (> $5,000)" row), rather than inventing a contact.
5. Re-test a Module 5 question (for example, looking up `PO-10234`) to confirm `po_lookup_tool`
   still works on this new agent version.
6. Type `exit` to end the session.

## ✅ Checkpoint / Validate

- [ ] `python client.py` prints the connected-tools line, then the agent-version confirmation,
  without import or authentication errors.
- [ ] Asking an escalation-contact question returns the correct contact and SLA from policy
  Section 8, not a guessed answer.
- [ ] Asking about the "no internal escalation" case (missing PO over $5,000) is handled
  correctly, not as a tool error.
- [ ] A Module 5 PO-lookup question and a Module 3/4 policy or trend question still work against
  this new agent version.

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

- **Client hangs or errors immediately after "Connected to MCP server..." never prints:** The
  subprocess FastMCP server likely failed to start. Confirm `python server.py` runs cleanly on
  its own (Step 4) and that `command="python"` in `StdioServerParameters` resolves to your
  activated venv's interpreter — launch `client.py` from the same terminal/venv you used to
  install `fastmcp` and `mcp`, not a different shell where `python` might point elsewhere.
- **`TypeError: object ... can't be used in 'await' expression` or similar async errors:** This
  usually means a sync call got left inside an `await`, or an `async def` got called without
  `await`/`asyncio.run`. Compare your `client.py` against Step 6–8 closely — the whole client body
  must be inside `async def main()`, invoked only via `asyncio.run(main())` at the bottom of the
  file, not called directly.
- **`RuntimeError: This event loop is already running` or nested `asyncio.run` errors:** Make sure
  you only call `asyncio.run(main())` once, at module level under `if __name__ == "__main__":`,
  and that you haven't also wrapped the script in another async runner.
- **Escalation-contact answers look wrong or stale:** Confirm `LAB_REPO_ROOT` in `.env` (set in
  Module 5) still points at the root of your local clone of *this* lab repository, so
  `server.py` reads the same `invoice-po-policy.md` you reviewed in Step 2.
- **Agent answers from memory instead of calling `get_escalation_contact`:** Confirm Step 5's
  instructions paragraph was actually added, and that the discrepancy-type phrase in your question
  reasonably matches a Section 8 row — an unrecognized phrase may lead the model to answer
  generically instead of calling the tool.
- **Module 3/4/5 questions stop working after this module:** This is the same failure mode called
  out in the Step 7 "Verify before running" box — your `tools=[...]` list didn't carry over
  `po_lookup_tool` and/or the File Search and Code Interpreter tool definitions from Module 5.
  Add them back into the same list and re-run `client.py`.
