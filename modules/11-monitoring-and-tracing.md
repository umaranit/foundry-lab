# Module 11: Monitoring & Tracing

**Objective:** Instrument the hosted **Invoice Discrepancy Assistant** with Azure Monitor
OpenTelemetry so every model call is traced automatically, wrap each test query in one
manually-created parent span carrying duration and token attributes, and read the resulting
cost/latency picture back in the Foundry portal — then confirm the Module 8 Agent Framework
agents show up in the same place with no comparable instrumentation effort.

**Estimated duration:** 40 minutes

**Prerequisites:** Modules 6–10 complete — the hosted **Invoice Discrepancy Assistant** has the
`lookup_po_by_number` function tool and the `get_escalation_contact` MCP tool (Module 6),
`agent_framework_workflow.py` runs successfully (Module 8), `discrepancy-eval-seed.jsonl` exists
(Module 10), and your `.env` file (`PROJECT_ENDPOINT`, `MODEL_DEPLOYMENT`, `LAB_REPO_ROOT`) and
virtual environment from Module 0 are still in place.

> **Scope note:** this module's telemetry bootstrap — `configure_azure_monitor` plus
> `OpenAIInstrumentor` fed by `project_client.telemetry.get_application_insights_connection_string()`
> — is verified against the actual `mslearn-genaiops/docs/05-monitoring-tracing.md` reference lab.
> Two things differ deliberately from that reference lab. First, that lab provisions a brand-new
> Application Insights resource via `azd up`; a Foundry project already has one connected the
> moment it's created, so this module's first step is to *confirm* that connection, not create
> it. Second, that lab's "view an individual trace tree" step runs a custom local script,
> `python src/tests/check_traces.py`, that queries Log Analytics directly and prints a nested
> tree in the terminal. This module doesn't reproduce that script — its exact code wasn't
> available to verify line-by-line — and instead uses the Foundry portal's own **Traces** view
> for that step. `check_traces.py` is mentioned once at the end as an optional, unverified
> advanced technique you can explore directly in the `mslearn-genaiops` repo, not something this
> module walks through.

## Steps

### 1. Reuse your working folder and install the tracing packages

1. Reactivate the virtual environment you've been using since Module 0:

   ```powershell
   Set-Location "$HOME\invoice-discrepancy-assistant"
   .\.venv\Scripts\Activate.ps1
   ```

2. Install the two packages this module adds on top of everything installed in Modules 5–8:

   ```powershell
   pip install azure-monitor-opentelemetry opentelemetry-instrumentation-openai-v2
   ```

   - `azure-monitor-opentelemetry` configures the OpenTelemetry SDK's global tracer provider with
     an exporter that sends spans to Application Insights.
   - `opentelemetry-instrumentation-openai-v2` patches the `openai` client library (the same
     `openai_client` you've been using since Module 5) so every `responses.create()` call
     automatically becomes its own span, with no code changes to your existing request logic.

### 2. Confirm your project already has Application Insights connected

1. Unlike the reference lab, you did **not** run `azd up` or provision anything for this module —
   a Foundry project has an Application Insights resource connected automatically the moment it's
   created, specifically so that `project_client.telemetry.get_application_insights_connection_string()`
   (used in Step 3) has something to return.
2. Confirm this in the Foundry portal: open your project, and in the left navigation select
   **Agents**, then select the **Traces** tab at the top. You should see either an empty trace
   list (if you haven't sent any instrumented requests yet — that's expected before Step 7) or a
   **Connect** prompt if no Application Insights resource is linked.
3. If you do see a **Connect** prompt instead of a trace list: select it, choose your project's
   existing Application Insights resource (or **Create new** if none exists), and confirm the
   connection before continuing. This is the one case where you'd do manual setup in this module,
   and it should be rare for a project created in Module 1.

### 3. Write `tracing_demo.py` — imports, environment, and the telemetry bootstrap

Create a file named `tracing_demo.py` in your working folder. The telemetry setup must run
**before** any model calls are made, so it goes at the very top of the script, immediately after
the imports:

```python
"""Trace Invoice Discrepancy Assistant runs with Azure Monitor OpenTelemetry.

Builds on Module 10's capture_responses.py conversation-per-query loop against the Module 6
hosted agent (including the local lookup_po_by_number dispatch and the MCP
get_escalation_contact dispatch). OpenAIInstrumentor auto-creates a child span for every
underlying model call with no extra code; this script additionally wraps each query in one
manually-created parent span carrying custom duration/token attributes, mirroring the reference
lab's per-test-prompt span pattern -- adapted here to our discrepancy-eval-seed.jsonl queries.
"""

import asyncio
import json
import os
import time
from pathlib import Path

from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.monitor.opentelemetry import configure_azure_monitor
from dotenv import load_dotenv
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from opentelemetry import trace
from opentelemetry.instrumentation.openai_v2 import OpenAIInstrumentor
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

from functions import lookup_po_by_number

load_dotenv()

project_endpoint = os.environ["PROJECT_ENDPOINT"]
AGENT_NAME = "Invoice Discrepancy Assistant"

DATA_DIR = Path(os.environ["LAB_REPO_ROOT"]) / "data"
SEED_FILE = DATA_DIR / "discrepancy-eval-seed.jsonl"
SAMPLE_SIZE = 3  # keep this demo quick; raise to len(seed rows) to trace the whole seed set

server_params = StdioServerParameters(command="python", args=["server.py"])

credential = DefaultAzureCredential()
project_client = AIProjectClient(endpoint=project_endpoint, credential=credential)

# --- Telemetry bootstrap: verified against mslearn-genaiops/docs/05-monitoring-tracing.md ---
connection_string = project_client.telemetry.get_application_insights_connection_string()
configure_azure_monitor(connection_string=connection_string)
OpenAIInstrumentor().instrument()

tracer = trace.get_tracer(__name__)
```

> **Why `project_client` and `credential` are created at module level, not inside `main()`:** the
> connection string has to be fetched and `configure_azure_monitor`/`OpenAIInstrumentor` have to
> run before the first `responses.create()` call anywhere in the process. Creating them here once
> and reusing `project_client` in Step 5 avoids opening a second, redundant `AIProjectClient`
> later in the script.

### 4. Load the sample queries and build the per-query traced request function

Append the seed-loading helper and the traced request function to `tracing_demo.py`. This reuses
Module 10's `capture_response` conversation-per-query loop almost unchanged — the difference is
the `tracer.start_as_current_span(...)` block wrapped around it, and the custom attributes
attached once the response comes back:

```python
def load_sample_rows(limit: int) -> list[dict]:
    with SEED_FILE.open(encoding="utf-8") as seed_file:
        rows = [json.loads(line) for line in seed_file if line.strip()]
    return rows[:limit]


async def run_traced_query(openai_client, functions_dict, index: int, query: str) -> None:
    """Send one query to the hosted agent inside one manually-created parent span.

    OpenAIInstrumentor (instrumented in Step 3) already wraps every responses.create() call in
    its own child span with no extra code here. This function additionally wraps the whole
    query -- including any function-call round trip -- in a parent span, so the Foundry portal's
    trace tree groups "this whole test query" together with the auto-instrumented model-call
    span(s) nested underneath it.
    """
    with tracer.start_as_current_span(f"invoice_query_{index}") as span:
        span.set_attribute("invoice.query", query)
        start_time = time.perf_counter()

        conversation = openai_client.conversations.create()
        openai_client.conversations.items.create(
            conversation_id=conversation.id,
            items=[{"type": "message", "role": "user", "content": query}],
        )

        input_list: ResponseInputParam = []
        response = openai_client.responses.create(
            conversation=conversation.id,
            input=input_list,
            extra_body={"agent_reference": {"name": AGENT_NAME, "type": "agent_reference"}},
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
                    FunctionCallOutput(type="function_call_output", call_id=item.call_id, output=result)
                )

            response = openai_client.responses.create(
                conversation=conversation.id,
                input=input_list,
                extra_body={"agent_reference": {"name": AGENT_NAME, "type": "agent_reference"}},
            )

        duration_s = time.perf_counter() - start_time
        span.set_attribute("response.duration_s", duration_s)

        # ⚠️ Verify before relying on this: whether `usage` is populated on a Responses API
        # object returned through `agent_reference` was not confirmed against an official example
        # at the time this module was written. Guard with getattr/None so a missing `usage` just
        # skips the token attributes instead of raising.
        usage = getattr(response, "usage", None)
        if usage is not None:
            span.set_attribute("response.total_tokens", usage.total_tokens)
            span.set_attribute("response.input_tokens", usage.input_tokens)
            span.set_attribute("response.output_tokens", usage.output_tokens)

        print(f"\n[{index}] QUERY: {query}")
        print(f"[{index}] DURATION: {duration_s:.2f}s")
        if usage is not None:
            print(
                f"[{index}] TOKENS: {usage.total_tokens} "
                f"(in {usage.input_tokens} / out {usage.output_tokens})"
            )
        print(f"[{index}] RESPONSE: {response.output_text}")
```

> **One conversation per query, same reason as Module 10:** each sample query should measure the
> agent answering cold, not building on a prior query's context, so this opens a brand-new
> `conversation` every call — identical to `capture_responses.py`.

### 5. Add the async driver and run it

Append the entry point:

```python
async def main() -> None:
    sample_rows = load_sample_rows(SAMPLE_SIZE)

    async with stdio_client(server_params) as (stdio, write):
        async with ClientSession(stdio, write) as session:
            await session.initialize()
            tools_response = await session.list_tools()

            def make_tool_func(tool_name):
                async def tool_func(**kwargs):
                    return await session.call_tool(tool_name, kwargs)

                tool_func.__name__ = tool_name
                return tool_func

            functions_dict = {t.name: make_tool_func(t.name) for t in tools_response.tools}

            with project_client.get_openai_client() as openai_client:
                for index, row in enumerate(sample_rows, start=1):
                    await run_traced_query(openai_client, functions_dict, index, row["query"])


if __name__ == "__main__":
    asyncio.run(main())
```

1. Run it:

   ```powershell
   python tracing_demo.py
   ```

2. Confirm you see three `[1]`/`[2]`/`[3]` blocks, each printing its query, a duration in seconds,
   a token line (if `usage` was populated), and the agent's response text. This is the same
   per-query console output style as Module 10's `capture_responses.py`, just for three rows
   instead of fifteen, and now instrumented.
3. Wait a minute or two for Application Insights ingestion — traces don't always appear
   instantly in the portal views in the next steps.

### 6. View aggregated metrics in the Foundry portal: Monitoring → Resource usage

1. In the Foundry portal, open your project.
2. In the left navigation, select **Monitoring**.
3. Select **Resource usage** to see a summary of model interactions.
4. Confirm you can see, for the three calls you just made:
   - **Total requests** — should include your three (plus any earlier Module 5–10 traffic if
     Application Insights was already connected then).
   - **Total token count**, **prompt/input token count**, and **completion/output token count** —
     compare these against the `[index] TOKENS: ...` lines your script printed.
   - A **latency** metric — compare against the `[index] DURATION: ...` lines your script
     printed.
5. If no data appears yet, wait a minute and refresh the page — this matches the reference lab's
   own note that ingestion isn't instantaneous.

### 7. View one individual trace/span tree in the Traces view

1. Still in the Foundry portal, in the left navigation select **Agents**, then select the
   **Traces** tab at the top.
2. Find one of the three traces you just generated (search or filter by time, or by Response ID
   if you noted one from the console output) and open it.
3. Confirm the span tree shows:
   - Your manually-created parent span, `invoice_query_1` (or `_2`/`_3`), with the
     `invoice.query`, `response.duration_s`, and (if populated) `response.total_tokens`
     attributes you set in Step 4.
   - One or more child spans created automatically by `OpenAIInstrumentor` underneath it, for
     each `responses.create()` call that query made — including a second one if that query
     triggered a function/tool call and a follow-up call with the tool's output.
4. This is the Foundry-portal equivalent of the reference lab's local
   `trail_guide_v1 → v1_day-hike-gear → chat gpt-5.1` nested tree — same parent/child shape
   (your custom query span, then the auto-instrumented model-call span beneath it), viewed in the
   portal instead of a terminal.

### 8. Confirm the Module 8 Agent Framework spans show up too

1. Recall the facilitator note from earlier planning: the Microsoft Agent Framework SDK used in
   Module 8 emits OpenTelemetry spans **natively** — every agent run and handoff already creates
   its own spans through the standard `opentelemetry` API, the same API this module used directly
   in Step 4. The only thing missing for those spans to reach Application Insights is an exporter
   pointed at your project's connection string, which is exactly what `configure_azure_monitor`
   sets up as the *global* tracer provider for the whole process — not just for this script.
2. Open your existing `agent_framework_workflow.py` from Module 8 and temporarily add the same
   two-line telemetry bootstrap near the top (after its existing imports, before the chat client
   and agents are constructed), reusing the same connection string call:

   ```python
   from azure.monitor.opentelemetry import configure_azure_monitor

   connection_string = project_client.telemetry.get_application_insights_connection_string()
   configure_azure_monitor(connection_string=connection_string)
   ```

   Note what's **not** here: no `OpenAIInstrumentor()` and no manually-created parent span. Those
   were only needed in `tracing_demo.py` because the hosted-agent path goes through the raw
   `openai` client; Agent Framework's own spans need nothing beyond the exporter.
3. Run it:

   ```powershell
   python agent_framework_workflow.py
   ```

4. Back in the Foundry portal's **Agents → Traces** tab (Step 7), refresh after a minute and
   confirm a new trace appears for this run, alongside the hosted-agent traces from Step 7 — same
   Application Insights resource, same view, with Agent Framework's own span names (for the
   workflow and the specialist agent it handed off to) instead of your custom `invoice_query_*`
   names.
5. Revert the two lines you added to `agent_framework_workflow.py` once you're done confirming
   this, so Module 8's file matches what that module describes again.

### 9. Connect what you're seeing back to Module 10

Module 10 ran these same kinds of queries through the built-in evaluators and produced a quality
score — `intent_resolution`, `relevance`, `groundedness` — per row. This module ran a small sample
of the same seed queries through the same hosted agent and produced a *cost and latency* profile
for each one instead. Neither view is complete on its own: a response can score well on
`groundedness` while taking 8 seconds and consuming several thousand tokens, or respond in under a
second while being ungrounded. Treat the Module 10 evaluation scores and this module's
duration/token attributes as two lenses on the same underlying runs, not two unrelated exercises —
a production monitoring setup would want both before shipping a prompt or agent change.

## ✅ Checkpoint / Validate

- [ ] `pip install azure-monitor-opentelemetry opentelemetry-instrumentation-openai-v2` succeeded
  in your Module 0 virtual environment.
- [ ] `python tracing_demo.py` ran to completion and printed a query, duration, and (if available)
  token usage for all three sample rows.
- [ ] The Foundry portal's **Monitoring → Resource usage** page shows request counts, token
  counts, and latency that are consistent with what the script printed.
- [ ] The Foundry portal's **Agents → Traces** tab shows an individual trace for one of your three
  sample queries, with your `invoice_query_N` parent span and its `response.duration_s`/
  `response.total_tokens` attributes, and at least one auto-instrumented model-call child span
  nested underneath it.
- [ ] After temporarily adding the two-line telemetry bootstrap to `agent_framework_workflow.py`
  and running it, a new trace for that run appears in the same **Traces** tab, confirming Module
  8's agents are traced with no `OpenAIInstrumentor`/manual-span work required.

## 🚀 Try it yourself

Raise `SAMPLE_SIZE` in `tracing_demo.py` to trace all 15 rows of `discrepancy-eval-seed.jsonl`
instead of 3, then add one more custom attribute to the parent span in `run_traced_query` —
for example `span.set_attribute("invoice.query_index", index)` or a boolean flag for whether the
query triggered any function call (`span.set_attribute("invoice.used_tool", bool(input_list))`).
Re-run the script, then in the Foundry portal's **Traces** tab, confirm your new attribute appears
on every parent span and use it to find the single slowest or most token-hungry query across the
full set — does it correspond to a query that needed a tool call (`lookup_po_by_number` or
`get_escalation_contact`), or to one that didn't?

## ⚠️ Troubleshooting

- **`project_client.telemetry.get_application_insights_connection_string()` returns `None` or
  raises an error:** Your project doesn't have an Application Insights resource connected yet —
  revisit Step 2 and use the portal's **Connect** prompt under **Agents → Traces** to link one.
- **No data in `Monitoring → Resource usage` or `Agents → Traces` after running the script:**
  Ingestion typically takes a minute or two; wait and refresh. If it's been several minutes,
  re-check that `configure_azure_monitor(connection_string=...)` ran (no exception at startup)
  and that the connection string came from the same project you're viewing in the portal.
- **`tracing_demo.py` hangs after printing nothing:** The same MCP-subprocess cause as Modules 6
  and 9 — confirm `python server.py` still runs cleanly on its own from the same activated-venv
  terminal.
- **Token attributes never show up on any parent span:** `usage` may not be populated on every
  Responses API object shape this lab uses — the `getattr(response, "usage", None)` guard in
  Step 4 means your script still runs and prints a duration either way; this is flagged as
  unverified in that step's callout rather than something to treat as broken.
- **You want the deeper local trace-tree script the reference lab uses:** that lab's
  `python src/tests/check_traces.py` queries Log Analytics directly via the `azure-monitor-query`
  SDK and prints a nested parent/child tree with inline duration and token attributes in the
  terminal, without needing the portal at all. This module doesn't reproduce it — adapting it
  correctly wasn't verified against this lab's data shapes — but if you want to go further, the
  real script is in the `mslearn-genaiops` repo at `src/tests/check_traces.py` and is a reasonable
  reference for building a local trace viewer for your own project.
- **Forgot to revert the two lines added to `agent_framework_workflow.py` in Step 8:** harmless to
  leave in (it just re-points the same exporter at the same resource every run), but remove them
  if you want that file to match Module 8's description exactly again.
