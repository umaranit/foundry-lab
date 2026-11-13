# Module 10: Evaluation

**Objective:** Capture real responses from the hosted **Invoice Discrepancy Assistant** against a
seed set of discrepancy queries, then run a Foundry cloud evaluation with the verified built-in
**intent_resolution**, **relevance**, and **groundedness** evaluators, and interpret the results.

**Estimated duration:** 45 minutes

**Prerequisites:** Modules 5–6 complete — the **Invoice Discrepancy Assistant** has a version with
the `lookup_po_by_number` function tool, the `get_escalation_contact` MCP tool, File Search, and
Code Interpreter, and your `.env` file (`PROJECT_ENDPOINT`, `MODEL_DEPLOYMENT`, `LAB_REPO_ROOT`)
and virtual environment from Module 0 are still in place.

> **Scope correction:** this module's evaluators and API shapes are verified against the actual
> `mslearn-genaiops/src/evaluators/evaluate_agent.py` reference script (the Evals API via
> `project_client.get_openai_client()`, not a standalone `evaluate` SDK call), not against
> invented evaluator names. That reference script's dataset already had each `response` baked in
> ahead of time — this lab's dataset does not, because the agent is live, so this module adds a
> preliminary **capture** step the reference script doesn't need. There is also no custom
> evaluator in this module — only the three verified built-in evaluators above.

## Steps

### 1. Review the seed dataset

1. Open [`data/discrepancy-eval-seed.jsonl`](/data/discrepancy-eval-seed.jsonl) in VS Code.
2. Note the shape: one JSON object per line, each with exactly two fields, `query` and
   `ground_truth` — for example:

   ```json
   {"query": "An invoice for $2,200 references PO-10243, but that PO number doesn't exist in our system. What do we need to do before this can be paid?", "ground_truth": "After confirming via the PO lookup tool that PO-10243 genuinely does not exist ..."}
   ```

3. Note the coverage: all four discrepancy types (price mismatch, quantity mismatch, duplicate
   invoice, missing PO) across their different tolerance tiers from
   [`invoice-po-policy.md`](/data/invoice-po-policy.md), plus two "trick" rows — one where the
   invoice matches the PO exactly (no discrepancy at all) and one that tests the Section 7
   approval-authority limits rather than a discrepancy tolerance — and one row that exercises the
   Module 6 `get_escalation_contact` tool directly.
4. **There is no `response` field yet.** A real evaluation needs a real captured answer, not just
   the expected one — that's what Step 3 produces.

> **Why 15 rows is enough for this lab, but wouldn't be for production:** the built-in evaluators
> report averages and pass rates, so a handful of rows is enough to see the mechanism work and
> spot an obviously broken answer, but far too small a sample to draw a confident conclusion
> about real-world quality. Treat the numbers you see in this module as "does the pipeline
> work and does anything look obviously wrong," not as a statistically meaningful quality score.

### 2. Decide what's under test (no new installs needed)

1. This module evaluates the Module 6 hosted **Invoice Discrepancy Assistant** — not the Module 8
   Agent Framework orchestrator. The hosted agent already has grounding and both tools, and the
   capture script in Step 3 reuses the Module 5/6 `conversations`/`responses` client pattern
   directly. Evaluating the Module 8 orchestrator instead would mean adapting the capture step to
   its different (non-`conversations`/`responses`) invocation pattern, which this module doesn't
   cover.
2. Every package this module needs — `azure-ai-projects`, `azure-identity`, `openai`,
   `python-dotenv`, `fastmcp`, and `mcp` — was already installed in Modules 5–6. Confirm your
   virtual environment from Module 0 is still activated and skip straight to Step 3:

   ```powershell
   Set-Location "$HOME\invoice-discrepancy-assistant"
   .\.venv\Scripts\Activate.ps1
   ```

### 3. Write the capture script

The seed file has no `response` field, and every built-in evaluator in this module needs one. This
script sends each seed `query` to the live hosted agent and records what it actually answers.

Create a file named `capture_responses.py` in your working folder:

```python
"""Capture real Invoice Discrepancy Assistant responses for evaluation.

Reads discrepancy-eval-seed.jsonl (query + ground_truth only), sends each query to the
hosted agent using the same conversations/responses client pattern as Modules 5-6 (including
the local lookup_po_by_number dispatch and the MCP get_escalation_contact dispatch added in
Module 6), and writes enriched query/response/ground_truth triples to
discrepancy-eval-dataset.jsonl -- the only file this module's evaluation step can read.
"""

import asyncio
import json
import os
from pathlib import Path

from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from openai.types.responses.response_input_param import FunctionCallOutput, ResponseInputParam

from functions import lookup_po_by_number

load_dotenv()

project_endpoint = os.environ["PROJECT_ENDPOINT"]
AGENT_NAME = "Invoice Discrepancy Assistant"

DATA_DIR = Path(os.environ["LAB_REPO_ROOT"]) / "data"
SEED_FILE = DATA_DIR / "discrepancy-eval-seed.jsonl"
DATASET_FILE = DATA_DIR / "discrepancy-eval-dataset.jsonl"

server_params = StdioServerParameters(command="python", args=["server.py"])
```

> **Why reference the agent by `AGENT_NAME` instead of calling `create_version` again?** This
> script only needs to *ask* the already-deployed agent questions — it doesn't need to change its
> configuration. `agent_reference` in Step 5 below accepts the agent's name directly, resolving to
> its latest version, so there's no need to redefine `PromptAgentDefinition` or its tools here.

### 4. Load the seed rows and capture one response per query

Append the row-reading helper and the per-query capture function to `capture_responses.py`:

```python
def load_seed_rows() -> list[dict]:
    with SEED_FILE.open(encoding="utf-8") as seed_file:
        return [json.loads(line) for line in seed_file if line.strip()]


async def capture_response(openai_client, functions_dict, agent_name: str, query: str) -> str:
    """Send one query to the hosted agent and return its final output_text.

    Structurally the same function-call handling loop as Module 6's client.py, but driven by
    a single query instead of an interactive input() loop, and using a fresh conversation per
    row so earlier seed rows can't leak context into later ones.
    """
    conversation = openai_client.conversations.create()

    openai_client.conversations.items.create(
        conversation_id=conversation.id,
        items=[{"type": "message", "role": "user", "content": query}],
    )

    input_list: ResponseInputParam = []
    response = openai_client.responses.create(
        conversation=conversation.id,
        input=input_list,
        extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
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
            extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
        )

    return response.output_text
```

> **One conversation per query, on purpose:** Modules 5–6's client keeps a single conversation
> alive for the whole interactive session, since a real analyst's follow-up questions should see
> prior context. An evaluation dataset is the opposite — each row needs to measure the agent
> answering a single query cold, so this function opens a brand-new `conversation` every call.

### 5. Run the capture loop and write the enriched dataset

Append the async entry point:

```python
async def main():
    seed_rows = load_seed_rows()
    print(f"Captured 0/{len(seed_rows)} responses", end="\r", flush=True)

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

            with (
                DefaultAzureCredential() as credential,
                AIProjectClient(endpoint=project_endpoint, credential=credential) as project_client,
                project_client.get_openai_client() as openai_client,
            ):
                enriched_rows = []
                for index, row in enumerate(seed_rows, start=1):
                    response_text = await capture_response(
                        openai_client, functions_dict, AGENT_NAME, row["query"]
                    )
                    enriched_rows.append(
                        {
                            "query": row["query"],
                            "response": response_text,
                            "ground_truth": row["ground_truth"],
                        }
                    )
                    print(f"Captured {index}/{len(seed_rows)} responses", end="\r", flush=True)

    with DATASET_FILE.open("w", encoding="utf-8") as dataset_file:
        for row in enriched_rows:
            dataset_file.write(json.dumps(row) + "\n")

    print(f"\nWrote {len(enriched_rows)} rows to {DATASET_FILE}")


if __name__ == "__main__":
    asyncio.run(main())
```

1. Run it:

   ```powershell
   python capture_responses.py
   ```

2. Confirm the counter reaches `Captured 15/15 responses` and the script prints
   `Wrote 15 rows to ...\data\discrepancy-eval-dataset.jsonl`.
3. Open [`data/discrepancy-eval-dataset.jsonl`](/data/discrepancy-eval-dataset.jsonl) (it's
   generated next to the seed file) and spot-check two or three rows — each should now have
   `query`, `response`, and `ground_truth`, with `response` containing the agent's real answer
   text, not a placeholder.

> **Re-run this script any time you change the agent.** The capture step has no memory of which
> agent version produced `discrepancy-eval-dataset.jsonl` — if you create another agent version
> after this (a new instructions tweak, an added tool), the evaluation in the rest of this module
> will silently score the *old* responses unless you re-run `capture_responses.py` first.

### 6. Write the evaluation script — upload the dataset

Create a file named `evaluate.py` in your working folder. Start with the imports, configuration,
and the dataset-upload step:

```python
import os
import sys
import time
from pathlib import Path

from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from openai.types.eval_create_params import DataSourceConfigCustom
from openai.types.evals.create_eval_jsonl_run_data_source_param import (
    CreateEvalJSONLRunDataSourceParam,
    SourceFileID,
)

load_dotenv()

project_endpoint = os.environ["PROJECT_ENDPOINT"]
model_deployment_name = os.environ["MODEL_DEPLOYMENT"]
dataset_name = "invoice-discrepancy-eval-dataset"
dataset_version = "1"

dataset_path = Path(os.environ["LAB_REPO_ROOT"]) / "data" / "discrepancy-eval-dataset.jsonl"

if not dataset_path.exists():
    print(f"ERROR: {dataset_path} not found. Run capture_responses.py first.")
    sys.exit(1)

project_client = AIProjectClient(endpoint=project_endpoint, credential=DefaultAzureCredential())
client = project_client.get_openai_client()


def upload_dataset() -> str:
    """Upload the captured dataset and return its dataset ID.

    Foundry rejects re-uploading the same name+version, so a re-run of this script reuses the
    existing dataset instead of failing.
    """
    try:
        data_id = project_client.datasets.upload_file(
            name=dataset_name,
            version=dataset_version,
            file_path=str(dataset_path),
        ).id
        print(f"Uploaded dataset. Dataset ID: {data_id}")
    except Exception as upload_error:
        if "already exists" in str(upload_error):
            print(f"Dataset {dataset_name} v{dataset_version} already exists — reusing it.")
            data_id = project_client.datasets.get(name=dataset_name, version=dataset_version).id
        else:
            raise
    return data_id
```

> **`model_deployment_name` here is the judge model, not a new deployment.** This lab deploys one
> chat model in Module 1 (`MODEL_DEPLOYMENT`), and this module reuses it both as the system under
> test (via the hosted agent in `capture_responses.py`) and as the judge model the built-in
> evaluators use to score responses. That's a fine simplification for this lab; in a production
> evaluation pipeline, judge and system-under-test models are often deliberately different.

### 7. Define the evaluation — item schema and the three built-in evaluators

Append the evaluation-definition step. This is the exact verified shape from
`mslearn-genaiops/src/evaluators/evaluate_agent.py`, with only the dataset/model names adapted:

```python
def create_evaluation_definition():
    """Register an evaluation definition: the dataset's item schema, plus which built-in
    evaluators to run and how to map dataset fields to each one's parameters."""
    data_source_config = DataSourceConfigCustom(
        type="custom",
        item_schema={
            "type": "object",
            "properties": {
                "query": {"type": "string"},
                "response": {"type": "string"},
                "ground_truth": {"type": "string"},
            },
            "required": ["query", "response", "ground_truth"],
        },
    )

    testing_criteria = [
        {
            "type": "azure_ai_evaluator",
            "name": "intent_resolution",
            "evaluator_name": "builtin.intent_resolution",
            "initialization_parameters": {"deployment_name": model_deployment_name},
            "data_mapping": {
                "query": "{{item.query}}",
                "response": "{{item.response}}",
            },
        },
        {
            "type": "azure_ai_evaluator",
            "name": "relevance",
            "evaluator_name": "builtin.relevance",
            "initialization_parameters": {"deployment_name": model_deployment_name},
            "data_mapping": {
                "query": "{{item.query}}",
                "response": "{{item.response}}",
            },
        },
        {
            "type": "azure_ai_evaluator",
            "name": "groundedness",
            "evaluator_name": "builtin.groundedness",
            "initialization_parameters": {"deployment_name": model_deployment_name},
            "data_mapping": {
                "query": "{{item.query}}",
                "response": "{{item.response}}",
                "context": "{{item.ground_truth}}",
            },
        },
    ]

    eval_object = client.evals.create(
        name="Invoice Discrepancy Assistant Quality Evaluation",
        data_source_config=data_source_config,
        testing_criteria=testing_criteria,
    )
    print(f"Created evaluation definition. Evaluation ID: {eval_object.id}")
    return eval_object
```

> **Why `groundedness` maps `context` to `{{item.ground_truth}}`:** `intent_resolution` and
> `relevance` only need `query`/`response` — they judge whether the agent understood and
> addressed the question. `groundedness` additionally needs something to check the response
> *against*; this dataset has no separate retrieved-context field, so the expected answer
> (`ground_truth`) doubles as that reference context. This is the same mapping the verified
> reference script uses. **This measures answer-vs-reference similarity, not true
> retrieval-groundedness in the strict sense:** the evaluator compares the answer with the
> expected answer, not with the actual policy text/citations the agent retrieved via File Search.
> This is a known simplification for this lab; a high score does not prove that the agent
> retrieved the right evidence or faithfully used it. A production evaluation pipeline should
> capture the agent's actual retrieved context/citations per query and supply that retrieved
> context to the groundedness evaluator, keeping `ground_truth` separate for reference-answer
> comparisons. The verified code and mapping remain unchanged for this exercise.

### 8. Start the run and poll for completion

Append the run-creation and polling steps:

```python
def run_evaluation(eval_object, data_id):
    """Start an asynchronous cloud evaluation run over every row in the dataset."""
    eval_run = client.evals.runs.create(
        eval_id=eval_object.id,
        name="invoice-discrepancy-baseline-eval",
        data_source=CreateEvalJSONLRunDataSourceParam(
            type="jsonl",
            source=SourceFileID(type="file_id", id=data_id),
        ),
    )
    print(f"Started run {eval_run.id} (status: {eval_run.status})")
    return eval_run


def poll_for_results(eval_object, eval_run):
    """Poll every 10 seconds until the run reaches a terminal status."""
    start_time = time.time()
    while True:
        run = client.evals.runs.retrieve(run_id=eval_run.id, eval_id=eval_object.id)
        elapsed = int(time.time() - start_time)

        if run.status == "completed":
            print(f"\nEvaluation completed in {elapsed} seconds")
            return run
        elif run.status == "failed":
            error_detail = getattr(run, "error", None) or "No additional details available."
            raise RuntimeError(
                f"Evaluation run failed after {elapsed}s.\n"
                f"  Eval ID: {eval_object.id}\n"
                f"  Run ID : {eval_run.id}\n"
                f"  Error  : {error_detail}\n"
                f"  Inspect further in the Foundry portal's Evaluations view."
            )
        else:
            print(f"  [{elapsed}s] Status: {run.status}", end="\r", flush=True)
            time.sleep(10)
```

> **This is an asynchronous cloud run.** `client.evals.runs.create(...)` returns immediately with
> a `queued` or `running` status — the actual scoring happens in the cloud. For 15 rows and three
> evaluators, expect this to take a few minutes; a larger dataset could take much longer.

### 9. Retrieve and interpret per-item scores

Append the results step:

```python
def retrieve_and_display_results(eval_object, run):
    """Fetch per-item evaluator outputs and print average scores and pass rates."""
    output_items = list(
        client.evals.runs.output_items.list(run_id=run.id, eval_id=eval_object.id)
    )

    errored_items = [item for item in output_items if getattr(item, "status", None) == "error"]
    scored_items = [item for item in output_items if getattr(item, "status", None) != "error"]

    if errored_items:
        print(f"\n{len(errored_items)} item(s) errored during evaluation.")
        print(f"First error: {getattr(errored_items[0], 'error', 'details unavailable')}")

    scores: dict[str, list[float]] = {"intent_resolution": [], "relevance": [], "groundedness": []}

    for item in scored_items:
        # Entries may come back as dicts or as model objects depending on SDK version.
        for result in getattr(item, "results", None) or []:
            if isinstance(result, dict):
                metric = result.get("metric") or result.get("name")
                value = result.get("score")
            else:
                metric = getattr(result, "metric", None) or getattr(result, "name", None)
                value = getattr(result, "score", None)
            if metric in scores and value is not None:
                scores[metric].append(float(value))

    print(f"\nTotal items : {len(output_items)}")
    print(f"Errored     : {len(errored_items)}")
    print(f"Scored      : {len(scored_items)}")
    print("\nAverage scores (1-5 scale, pass threshold >= 3):")
    for metric, values in scores.items():
        if values:
            average = sum(values) / len(values)
            pass_rate = sum(1 for v in values if v >= 3) / len(values) * 100
            print(f"  {metric:18s}: avg {average:.2f}  (pass rate {pass_rate:.0f}%, n={len(values)})")
        else:
            print(f"  {metric:18s}: no scores returned — check the Foundry portal's Evaluations view")

    return output_items


def main():
    data_id = upload_dataset()
    eval_object = create_evaluation_definition()
    eval_run = run_evaluation(eval_object, data_id)
    run = poll_for_results(eval_object, eval_run)
    retrieve_and_display_results(eval_object, run)


if __name__ == "__main__":
    main()
```

### 10. Run the evaluation

1. Run the script:

   ```powershell
   python evaluate.py
   ```

2. Confirm you see, in order: a dataset upload confirmation, an evaluation-definition creation
   confirmation, a run-started confirmation, a series of `Status: ...` polling lines ending in
   `Evaluation completed in ... seconds`, and finally the average-score summary for all three
   evaluators.
3. In the Foundry portal, open your project and go to **Evaluations**. Find **Invoice Discrepancy
   Assistant Quality Evaluation** and open the **invoice-discrepancy-baseline-eval** run.
4. Sort or filter by evaluator score in the portal's results grid and look at the lowest-scoring
   row for each evaluator — read the actual captured `response` against its `ground_truth` and
   form your own opinion on whether the score looks right.

### 11. Connect low scores back to earlier modules

A low score on a specific row or evaluator is a pointer back to a specific earlier module, not
just a number to note and move past:

- Low **intent_resolution** on a specific discrepancy type → revisit the agent's instructions
  (Module 2) for that case — the model may be misunderstanding what's being asked.
- Low **relevance** → the agent may be answering a nearby but different question (for example,
  explaining a policy section generally instead of applying it to the specific numbers in the
  query) — again, usually an instructions issue (Module 2).
- Low **groundedness** → in this lab's mapping, the answer differs from the reference answer.
  Inspect the reference, the policy, actual retrieval evidence (Module 3), and tool outputs
  (Modules 5–6) before attributing the score to a retrieval or tool failure; this score alone
  does not measure whether the answer is supported by the context actually retrieved.

## ✅ Checkpoint / Validate

- [ ] `python capture_responses.py` ran to completion and produced
  `discrepancy-eval-dataset.jsonl` with 15 fully populated `query`/`response`/`ground_truth` rows.
- [ ] `python evaluate.py` reached `status: "completed"` for its run, with 0 errored items (or, if
  any errored, you've opened the Foundry portal and understood why).
- [ ] The console summary shows average scores and pass rates for all three evaluators —
  `intent_resolution`, `relevance`, and `groundedness`.
- [ ] You can find the same run and its per-item scores in the Foundry portal's **Evaluations**
  view, and you've read at least one low-scoring row's actual response against its ground truth.

## 🚀 Try it yourself

Add two new rows (`query` + `ground_truth`) to `discrepancy-eval-seed.jsonl` targeting an edge
case you think the agent might get wrong — for example, a price mismatch where the percentage and
dollar thresholds disagree on which tier applies (re-read policy Section 3.1's "use the higher
tier" rule), or a duplicate-invoice case where the vendor's invoice number differs only by a
leading zero. Re-run `capture_responses.py` to pick up their real captured responses (this
regenerates the whole `discrepancy-eval-dataset.jsonl`, not just the two new rows), bump
`dataset_version` to `"2"` in `evaluate.py`, and re-run `evaluate.py`. Compare the new run's scores
against the baseline run in the Foundry portal.

## ⚠️ Troubleshooting

- **`discrepancy-eval-dataset.jsonl` not found when running `evaluate.py`:** Run
  `capture_responses.py` first — the evaluation script intentionally refuses to run against a
  dataset that doesn't exist yet rather than silently evaluating stale or missing data.
- **Dataset "already exists" errors on re-upload:** Expected on a second run with the same
  `dataset_version` — `upload_dataset()` catches this and calls
  `project_client.datasets.get(name=..., version=...)` instead of re-uploading. If you want a
  fresh upload instead of reusing the old one, bump `dataset_version`.
- **`item_schema`/`data_mapping` mismatches:** If `testing_criteria` references a field (for
  example, `{{item.context}}`) that doesn't exist in `item_schema` or in your JSONL rows, the run
  will error per-item rather than failing immediately — check `errored_items` in the console
  output and the Foundry portal's Evaluations view for the specific error.
- **Runs stuck in `"queued"`/`"running"` far longer than expected:** This is normal under shared
  or constrained quota; the reference script notes 15–60+ minutes isn't unusual for larger
  datasets. For this lab's 15 rows, a run stuck for more than several minutes is worth checking in
  the portal.
- **Runs that reach `"failed"` status:** Inspect `run.error` (printed by `poll_for_results`) and
  the Foundry portal's Evaluations view for the underlying cause — a common one is the judge
  model deployment (`model_deployment_name`) not matching what's actually deployed in your
  project (double check against Module 1's deployment name).
- **Forgetting to re-run `capture_responses.py` after changing the agent:** The evaluation only
  ever scores whatever is already in `discrepancy-eval-dataset.jsonl`. If you create a new agent
  version (a different instructions tweak, a new tool) and re-run `evaluate.py` without first
  re-running `capture_responses.py`, you are scoring the *previous* version's answers.
- **`capture_responses.py` hangs after printing nothing:** The same MCP-subprocess cause as
  Module 6 — confirm `python server.py` still runs cleanly on its own, and that you're running
  `capture_responses.py` from the same activated-venv terminal where `fastmcp` and `mcp` are
  importable.
