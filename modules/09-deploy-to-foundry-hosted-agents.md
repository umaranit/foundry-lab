# Module 9: Deploy to Foundry Hosted Agents

**Objective:** Package a simplified, single price-mismatch specialist from Module 8, run it
through the Agent Framework Responses hosting adapter, and deploy its source to Foundry's
managed Hosted Agent runtime. Wait for an active version, temporarily route traffic to it,
invoke it, then restore the previous endpoint configuration when present and delete the
temporary version.

**Estimated duration:** 45 minutes

**Prerequisites:** [Module 8](08-multi-agent-orchestration-agent-framework.md)'s working local
agent and [Module 1](01-foundry-project-and-model-setup.md)'s existing Foundry project and model
deployment. You also need:

- Python **3.13.x** locally, continuing with the same Python installation from
  [Module 0](00-prerequisites.md) through Module 9; no second local Python version is needed.
  The hosted runtime's `runtime="python_3_14"` in `CodeConfiguration` is a property of Foundry's
  remote container build, independent of the local interpreter used to author and test
  `main.py` via `ResponsesHostServer`.
- Azure CLI sign-in and the **Foundry Project Manager** role at project scope. Confirm your
  project supports Hosted Agents in its region and has available compute/model capacity.
- Application Insights connected to the project and permission to read cloud traces. Complete
  [Connect Application Insights for agent tracing](https://learn.microsoft.com/azure/foundry/tutorials/quickstart-create-foundry-resources#connect-application-insights-for-agent-tracing)
  before the first remote invocation; do not wait until Module 11.
- Permission to upload the lab's fictional PO data. Do not substitute real financial records
  or upload credentials.

## Why this module?

- **Where we are:** Module 8's code-first agents and tools run in a local Python process.
- **What's missing:** Successful local execution does not show how to package custom agent
  code, run it on managed compute, or invoke it through a cloud agent endpoint.
- **What we'll add:** A hosting entry point and source deployment for **one price-mismatch
  specialist**, followed by activation checks, temporary routing, invocation, and cleanup.
  The full multi-agent handoff workflow is deliberately outside this module's scope.
- **Where it runs:** You author and test locally, then Foundry remotely builds and runs the
  specialist and its packaged CSV lookup in a managed Hosted Agent container. This is the
  **Hosted Agent** kind, distinct from the earlier Prompt Agent; it is not Azure Functions.
- **By the end:** You can demonstrate a cloud PO price-comparison response and complete the
  temporary deployment's cleanup. Module 10 returns to the earlier Prompt Agent, so it does
  not require this temporary specialist version to remain running.

**Verified references, checked 2026-11-13:**

- [Deploy your first hosted agent — Python SDK path](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent?pivots=python):
  source ZIP, `create_version_from_code`, activation polling, endpoint routing, invocation,
  and temporary-version cleanup.
- [Host Microsoft Agent Framework agents as Foundry hosted agents](https://learn.microsoft.com/azure/foundry/how-to/develop/framework-hosted-agents?pivots=programming-language-python):
  `agent_framework_foundry_hosting.ResponsesHostServer`, `FoundryChatClient`, and
  `default_options={"store": False}`.
- [Hosted agent permissions](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agent-permissions)
  and [Hosted agents concepts](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents).

> **Scope guardrail:** This module deploys **only the single `price_mismatch_agent`**, with
> Module 8's instructions and `lookup_po_by_number` re-exposed through `@tool`. It does **not**
> deploy the coordinator, the other specialists, or the full `HandoffBuilder` workflow.
> Hosting that full multi-agent handoff workflow, including its handoff/result handling, is
> **unverified and left as an advanced follow-on**. This module does not claim to demonstrate it.
> Keep Module 8's local script and Modules 5–7's clients unchanged.

## Steps

### 1. Understand what moves into the cloud

| Aspect | Modules 2–7: Prompt Agent | Module 8: local Agent Framework | Module 9: Hosted Agent |
|---|---|---|---|
| Definition | `PromptAgentDefinition`: instructions, model, tools | Python agents and a `HandoffBuilder` workflow | `HostedAgentDefinition`: custom code, compute, runtime, protocol |
| Execution | Foundry runs the Prompt Agent; the client executes local function tools | Your laptop runs orchestration and tools, calling a Foundry model | Managed container compute runs the single specialist and its CSV tool |
| Deployment payload | Prompt/tool configuration | No application-code deployment | Source ZIP with an HTTP service entry point, dependencies, and PO data |
| Policy grounding | Tools configured in earlier modules | Policy embedded in specialist instructions | The same price-mismatch instructions, not File Search or Foundry IQ |

The Prompt Agent uses service-managed prompt execution, **not** the Hosted Agent kind.
Here the CSV lookup executes inside the hosted container; Module 6's
local function dispatcher and MCP server are not involved.

The deployment uses **remote build**, not a locally built Docker image. It does not create a
Foundry project or model deployment. Hosted compute is billable separately from model usage,
and must be cleaned up even if the request or provisioning fails.

### 2. Create an isolated working folder and configure settings

1. In your workshop working folder, create a **new** folder, separate from Module 8's script.
   The following commands are PowerShell:

   ```powershell
   New-Item -ItemType Directory -Path .\module09-hosted
   Set-Location .\module09-hosted
   New-Item -ItemType Directory -Path .\src
   python --version
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   az login
   az account show --output table
   ```

   Verify that the Python version is **3.13.x** before creating the isolated virtual environment,
   using the same Python installation as Module 0. The remote runtime is explicitly
   `python_3_14`; it does not require a matching local interpreter or a second Python installation.

2. Copy the existing fictional [po-records.csv](../data/po-records.csv) into the source folder.
   Replace the placeholder with your lab clone's root:

   ```powershell
   Copy-Item -LiteralPath "C:\path\to\your\local\clone\data\po-records.csv" `
       -Destination .\src\po-records.csv
   ```

3. Create `.env` in `module09-hosted`, **outside** `src`:

   ```text
   FOUNDRY_PROJECT_ENDPOINT=<copy the project endpoint from Foundry Home>
   FOUNDRY_MODEL_NAME=<your existing model deployment name>
   FOUNDRY_HOSTED_AGENT_NAME=invoice-price-hosted-<your-unique-suffix>
   ```

   Reuse Module 8's `PROJECT_ENDPOINT` and `MODEL_DEPLOYMENT` **values**, under the names above.
   Copy the actual endpoint; do not construct one from a resource name. Choose a unique hosted
   agent name that belongs only to you. **Do not use the Invoice Discrepancy Assistant's name
   or an agent with other users' traffic.** Routing in this exercise affects the entire named
   agent endpoint, not just your session.

4. Create `src\requirements.txt`:

   ```text
   agent-framework
   agent-framework-foundry-hosting
   azure-ai-projects>=2.3.0
   azure-identity
   python-dotenv
   pydantic
   ```

   Install into the new virtual environment, leaving Module 8's environment untouched:

   ```powershell
   python -m pip install -r .\src\requirements.txt
   python -m pip check
   ```

   > **⚠️ Verify against current docs:** The package names and adapter calls above are confirmed
   > in the linked hosting guide. Agent Framework and Hosted Agents evolve quickly: check the
   > current guide and Python SDK quickstart before running this module. The source-deployment
   > APIs require `azure-ai-projects>=2.3.0`; older packages may not export these models or
   > `create_version_from_code`. If a package requires a prerelease, use the current documented
   > installation instructions rather than guessing an adapter API.

### 3. Implement the single-specialist service entry point

Create `src\main.py` with the following complete content. This keeps Module 8's lookup and
price-mismatch instructions, but replaces `LAB_REPO_ROOT` with a path relative to the packaged
entry point. `DefaultAzureCredential` replaces the laptop-only `AzureCliCredential` assumption.

```python
import csv
import os
from pathlib import Path
from typing import Annotated

from agent_framework import Agent, tool
from agent_framework.foundry import FoundryChatClient
from agent_framework_foundry_hosting import ResponsesHostServer
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv
from pydantic import Field

load_dotenv()

PO_RECORDS_FILE = Path(__file__).resolve().parent / "po-records.csv"


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


def main() -> None:
    if not PO_RECORDS_FILE.is_file():
        raise FileNotFoundError(f"Packaged PO records not found: {PO_RECORDS_FILE}")

    with DefaultAzureCredential() as credential:
        client = FoundryChatClient(
            project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
            model=os.environ["FOUNDRY_MODEL_NAME"],
            credential=credential,
        )

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
            default_options={"store": False},
        )

        server = ResponsesHostServer(price_mismatch_agent)
        server.run()


if __name__ == "__main__":
    main()
```

The hosting guide uses `AZURE_AI_MODEL_DEPLOYMENT_NAME`. This entry point explicitly reads
`FOUNDRY_MODEL_NAME` instead, matching the Python deployment quickstart; the deployment script
passes that same variable. There is no implicit fallback between the two names.

`ResponsesHostServer` serves the Responses protocol on port **8088** by default. It stays
running to accept requests; simply renaming Module 8's batch script to `main.py` would not do
that. `default_options={"store": False}` follows the guide because the hosting infrastructure
manages conversation history.

Locally, `DefaultAzureCredential` can use your Azure CLI sign-in. In Foundry it uses the
platform-provided identity environment; do not upload Azure CLI caches, API keys, or a service
principal secret to make cloud authentication work. Deployment permissions for your user and
model-access permissions for the runtime identity are separate concerns; follow the
[agent identity guidance](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity)
and permissions reference if runtime model access is denied.

> **Specialist limits:** There is no File Search, Foundry IQ, MCP integration, or handoff here.
> Policy Section 3 is embedded in the instructions. The reused lookup's missing-PO message
> does not add a Section 6 specialist; do not treat an answer about missing POs as a validated
> workflow. The tool reads fictional data and recommends actions; it does not approve payments
> or open Procurement cases.

### 4. Validate the hosting adapter locally before uploading

1. From `module09-hosted`, with its environment active, start the HTTP host:

   ```powershell
   $env:PYTHONNOUSERSITE = "1"
   python .\src\main.py
   ```

2. In a second PowerShell terminal, send this fictional invoice query. The single-quoted string
   preserves the dollar amount:

   ```powershell
   $body = @{
       input = 'Invoice INV-HOST-001 from Litware Manufacturing references PO-10234 for 1,000 CNC-machined aluminum brackets at $15.40 each. Look up the PO, calculate the price variance and total line-item impact, then state the required action and SLA.'
       stream = $false
   } | ConvertTo-Json

   $result = Invoke-RestMethod `
       -Uri http://localhost:8088/responses `
       -Method Post `
       -Body $body `
       -ContentType "application/json"
   $result | ConvertTo-Json -Depth 20
   ```

3. Validate the answer against the actual row in [po-records.csv](../data/po-records.csv):

   | Check | Expected result |
   |---|---|
   | PO and vendor | `PO-10234`, Litware Manufacturing |
   | Authoritative PO unit price | `$14.80` |
   | Invoiced unit price and quantity | `$15.40`, `1,000` |
   | Unit-price difference | `$15.40 - $14.80 = $0.60` |
   | Percentage variance | `$0.60 / $14.80 × 100 ≈ 4.05%` |
   | Total line-item impact | `$0.60 × 1,000 = $600` |
   | Required tier | **Escalate to Procurement**, because `$600 > $250`, despite percentage being below 5% |
   | Action and SLA | Hold payment; open a price-discrepancy case with the Procurement category manager; response within **3 business days**; no payment on the disputed line until confirmation |

   Wording can vary; these facts and the higher-tier decision must not. Confirm
   `lookup_po_by_number` execution in available tool-call traces/logs, not just plausible prose.
   The protocol response alone may not expose every internal tool call. Use a debugger
   breakpoint in the lookup for local evidence if needed.

4. Stop your local host with **Ctrl+C**. After a successful local test, capture the isolated
   environment's resolved dependency versions for the remote build:

   ```powershell
   python -m pip freeze | Set-Content -Path .\src\requirements.txt -Encoding utf8
   ```

   Inspect the manifest before uploading: it must contain package requirements, not local
   paths, editable installs, or authenticated package URLs. Pinning improves repeatability.
   Local testing on Python 3.13 validates the agent logic; remote build compatibility is a
   separate, server-side concern checked via the build logs (see **Remote build fails** in
   Troubleshooting).

### 5. Package, deploy, invoke, and clean up with the Python SDK

Create `deploy_hosted_agent.py` in `module09-hosted`, **not** in `src`. The layout should now be:

```text
module09-hosted/
  .env                      # local settings; never uploaded
  .venv/                    # local environment; never uploaded
  deploy_hosted_agent.py     # local deployment client; never uploaded
  src/
    main.py
    requirements.txt
    po-records.csv
```

The following is a complete deployment script. It uses the verified quickstart API shape,
with a strict three-file ZIP allowlist, a uniquely allocated temporary directory, and nested
cleanup so a restoration error does not skip the deletion attempt.

```python
import os
import tempfile
import time
import zipfile
from pathlib import Path

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    AgentEndpointConfig,
    CodeConfiguration,
    CodeDependencyResolution,
    FixedRatioVersionSelectionRule,
    HostedAgentDefinition,
    ProtocolConfiguration,
    ProtocolVersionRecord,
    ResponsesProtocolConfiguration,
    VersionSelector,
)
from azure.core.exceptions import ResourceNotFoundError
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv

WORKING_DIR = Path(__file__).resolve().parent
load_dotenv(WORKING_DIR / ".env")

endpoint = os.environ["FOUNDRY_PROJECT_ENDPOINT"]
model_name = os.environ["FOUNDRY_MODEL_NAME"]
agent_name = os.environ["FOUNDRY_HOSTED_AGENT_NAME"]
source_dir = WORKING_DIR / "src"

TEST_QUERY = (
    "Invoice INV-HOST-001 from Litware Manufacturing references PO-10234 "
    "for 1,000 CNC-machined aluminum brackets at $15.40 each. "
    "Look up the PO, calculate the price variance and total line-item impact, "
    "then state the required action and SLA."
)


def create_code_zip(source: Path, destination: Path) -> None:
    included = ("main.py", "requirements.txt", "po-records.csv")
    for name in included:
        path = source / name
        if path.is_symlink() or not path.is_file():
            raise ValueError(f"Expected a regular source file, not a link: {path}")

    with zipfile.ZipFile(destination, "w", zipfile.ZIP_DEFLATED) as archive:
        for name in included:
            archive.write(source / name, arcname=name)

    with zipfile.ZipFile(destination) as archive:
        names = archive.namelist()
        if set(names) != set(included) or len(names) != len(included):
            raise RuntimeError(f"Unexpected ZIP contents: {names}")
        print(f"ZIP contents: {', '.join(names)}", flush=True)


def wait_for_active_version(project_client: AIProjectClient, version: str) -> None:
    for attempt in range(60):
        time.sleep(10)
        details = project_client.agents.get_version(
            agent_name=agent_name,
            agent_version=version,
        )
        status = details["status"]
        print(f"Provisioning status: {status} (attempt {attempt + 1}/60)", flush=True)
        if status == "active":
            return
        if status == "failed":
            raise RuntimeError(f"Hosted agent provisioning failed: {dict(details)}")
    raise TimeoutError("Timed out waiting for the hosted agent version to become active.")


def confirm_version_deleted(project_client: AIProjectClient, version: str) -> None:
    for attempt in range(12):
        try:
            project_client.agents.get_version(
                agent_name=agent_name,
                agent_version=version,
            )
        except ResourceNotFoundError:
            print(f"Confirmed deletion of hosted agent version {version}", flush=True)
            return
        time.sleep(5)
    raise TimeoutError(
        f"Deletion not confirmed for {agent_name} version {version}. "
        "Check the version in Foundry before leaving the lab."
    )


def main() -> None:
    print(f"Project: {endpoint}\nDedicated lab agent: {agent_name}", flush=True)
    with tempfile.TemporaryDirectory(prefix="module09-hosted-") as temp_dir:
        code_zip = Path(temp_dir) / "source.zip"
        create_code_zip(source_dir, code_zip)

        with (
            code_zip.open("rb") as code_stream,
            DefaultAzureCredential() as credential,
            AIProjectClient(endpoint=endpoint, credential=credential) as project_client,
        ):
            created = None
            original_agent_endpoint = None
            try:
                created = project_client.agents.create_version_from_code(
                    agent_name=agent_name,
                    description="Temporary Contoso Finance price-mismatch specialist.",
                    definition=HostedAgentDefinition(
                        cpu="0.5",
                        memory="1Gi",
                        code_configuration=CodeConfiguration(
                            runtime="python_3_14",
                            entry_point=["python", "main.py"],
                            dependency_resolution=CodeDependencyResolution.REMOTE_BUILD,
                        ),
                        environment_variables={
                            "FOUNDRY_PROJECT_ENDPOINT": endpoint,
                            "FOUNDRY_MODEL_NAME": model_name,
                            "PYTHONNOUSERSITE": "1",
                        },
                        protocol_versions=[
                            ProtocolVersionRecord(protocol="responses", version="2.0.0"),
                        ],
                    ),
                    code=code_stream,
                )
                print(f"Created hosted agent version {created.version}", flush=True)
                wait_for_active_version(project_client, created.version)

                original_agent_endpoint = project_client.agents.get(
                    agent_name=agent_name,
                ).agent_endpoint
                print(f"Original endpoint configuration: {original_agent_endpoint}", flush=True)

                project_client.agents.update_details(
                    agent_name=agent_name,
                    agent_endpoint=AgentEndpointConfig(
                        version_selector=VersionSelector(
                            version_selection_rules=[
                                FixedRatioVersionSelectionRule(
                                    agent_version=created.version,
                                    traffic_percentage=100,
                                ),
                            ],
                        ),
                        protocol_configuration=ProtocolConfiguration(
                            responses=ResponsesProtocolConfiguration(),
                        ),
                    ),
                )
                print(f"Routed 100% of lab agent traffic to {created.version}", flush=True)

                with project_client.get_openai_client(agent_name=agent_name) as openai_client:
                    response = openai_client.responses.create(input=TEST_QUERY)
                if not response.output_text:
                    raise RuntimeError(f"Hosted invocation returned no text: {response}")
                print(f"Agent response:\n{response.output_text}", flush=True)
            finally:
                try:
                    if original_agent_endpoint is not None:
                        project_client.agents.update_details(
                            agent_name=agent_name,
                            agent_endpoint=original_agent_endpoint,
                        )
                        print("Original agent endpoint restored", flush=True)
                    elif created is not None:
                        print(
                            "No saved endpoint configuration to restore; "
                            "deleting the temporary version.",
                            flush=True,
                        )
                finally:
                    if created is not None:
                        print(
                            f"Deleting {agent_name} version {created.version}",
                            flush=True,
                        )
                        project_client.agents.delete_version(
                            agent_name=agent_name,
                            agent_version=created.version,
                            force=True,
                        )
                        confirm_version_deleted(project_client, created.version)


if __name__ == "__main__":
    main()
```

Before running, understand the lifecycle:

1. **ZIP the source.** Only `main.py`, `requirements.txt`, and `po-records.csv` are included,
   at the archive root, without a containing `src` directory. This allowlist excludes `.env`
   and its variants, credentials files, `.venv`, `.git`, `__pycache__`, and the deployment script,
   even if accidentally placed under `src`. Review the contents of the three allowed files
   too: filenames alone cannot prevent a secret pasted into source from being uploaded.
2. **Create a version.** `create_version_from_code` uploads the ZIP stream. Foundry resolves the
   dependency manifest remotely and starts `python main.py` with **0.5 CPU / 1 GiB memory**
   on **Python 3.14**, advertising **Responses 2.0.0**. `PYTHONNOUSERSITE="1"` follows the
   quickstart's dependency-isolation setting. No local `.env` is uploaded; only the explicitly
   listed, non-secret environment values are passed.
3. **Wait for readiness.** The script calls `agents.get_version` after each 10-second wait,
   up to 60 attempts (about 10 minutes plus API-call time). Only `"active"` proceeds.
   `"failed"` raises immediately with details; other statuses keep polling until timeout.
   A returned version number alone does not establish readiness.
4. **Save and route.** After activation, save the named agent's original `agent_endpoint`;
   configure a `VersionSelector` with a `FixedRatioVersionSelectionRule` sending **100%**
   of traffic to the new version and enable the Responses protocol. Do not run competing
   deployments against this agent while the script is running.
5. **Invoke the named agent.** `get_openai_client(agent_name=agent_name)` targets the hosted
   agent endpoint. `responses.create(input=TEST_QUERY)` is not a direct model request and does
   not invoke Module 8's local workflow. The hosted specialist calls the model and executes
   the packaged lookup itself.
6. **Clean up in `finally`.** Restore the original endpoint when one was saved, then delete
   **only the created version**, with `force=True`. Nested `finally` ensures deletion is
   attempted even if restoration raises. Errors remain visible; deletion is confirmed only
   when `get_version` returns not found. Temporary local ZIP files are removed on context exit.

> **New-agent endpoint caveat:** If the agent had no prior endpoint configuration, there is
> nothing to restore. This follows the official quickstart's `None` handling; it does not
> assume that `update_details(agent_endpoint=None)` clears routing. After deletion, the
> dedicated agent record or endpoint configuration may remain without a runnable version.
> That is not a persistent working deployment. Inspect it in Foundry; this module claims
> temporary-version cleanup, not deletion of the agent record or every project resource.

### 6. Run the cloud round trip and validate the response

1. Confirm the local host is stopped, your virtual environment is active, and `.env` names
   your **dedicated** lab agent. From `module09-hosted`, run:

   ```powershell
   python .\deploy_hosted_agent.py
   ```

2. Observe the ZIP's three filenames, the created version number, provisioning status, traffic
   update, and response. Record the **agent name and version** immediately for recovery if the
   terminal is interrupted. Keep the printed original endpoint configuration until cleanup
   is confirmed.
3. Compare the hosted answer with **every expected fact in Step 4's table**, especially the
   `$600` impact and Procurement tier. Inspect available hosted traces for the PO lookup.
   `"active"` validates startup, not policy correctness or tool execution; a fluent but
   unsupported answer is not a successful lab result.
4. Let the script finish. Do not close the terminal after seeing the answer: cleanup happens
   immediately afterward. Expect `Original agent endpoint restored` if a prior configuration
   existed, followed by `Confirmed deletion of hosted agent version <version>`.

The response is a sample outcome, not a guarantee of deterministic model behavior. If the
answer misses a threshold, first verify the instructions, actual tool result, and invoice
quantity before changing prompts or trying another deployment.

### 7. Confirm cleanup, including failure paths

1. In Foundry, open your dedicated hosted agent and refresh its versions. Confirm the temporary
   version is gone. If a prior endpoint configuration existed, confirm the prior version
   selection/protocol configuration is restored.
2. Verify that the existing **Invoice Discrepancy Assistant** Prompt Agent and Module 8's
   local script are unchanged.
3. If provisioning fails or times out after creation returns, or invocation fails, the same
   `finally` block attempts cleanup. A cleanup exception is a **failed lab run**, not success:
   keep the traceback and agent/version identifiers, restore prior routing if necessary, and
   remove the temporary version through Foundry's version management. Confirm its absence.
4. If the process is killed, the machine shuts down, or creation reaches the service but its
   response is lost, Python cannot guarantee cleanup. Inspect the dedicated agent for newly
   created versions before retrying; the script might not have received an ID to delete.
   Do not assume closing the client stops cloud compute.

> **Cost boundary:** Each Hosted Agent version is billable container compute, distinct from
> the model and other project resources. Clean up this version **now**, not at the end of the
> workshop. A prior hosted version, if any, is intentionally preserved and may still incur
> charges. Model usage, Application Insights, and other resources are not removed here.
> [Module 12](12-wrapup-governance.md)'s final resource-group cleanup is a separate, broader
> action; do not delete the shared project/resource group during this module.

[Module 10: Evaluation](10-evaluation.md) still targets the Module 6 Prompt Agent, and
[Module 11: Monitoring & tracing](11-monitoring-and-tracing.md) retains its existing scope.
Neither requires leaving this temporary Hosted Agent version running.

## ✅ Checkpoint / Validate

> Before moving on, confirm all of the following:
>
> - [ ] I can distinguish Prompt Agent configuration, local Agent Framework execution, and
>   managed Hosted Agent compute.
> - [ ] My payload contains only the single price-mismatch specialist, its `@tool` lookup,
>   dependency manifest, and fictional PO data — not the `HandoffBuilder` workflow.
> - [ ] The service ran through `ResponsesHostServer` locally, and the ZIP excluded secrets,
>   `.env`, `.venv`, `.git`, caches, and the deployment client.
> - [ ] The hosted version reached `"active"` before routing and invocation.
> - [ ] The named-agent response used the `$14.80` PO price and correctly escalated the
>   `$600` impact to Procurement with a 3-business-day SLA; I checked lookup execution evidence.
> - [ ] Prior endpoint configuration was restored when present, and temporary-version deletion
>   was confirmed. Any failure or uncertain cleanup has been resolved.
> - [ ] Earlier modules' Prompt Agent, clients, and local workflow remain unchanged.

## 🚀 Try it yourself

Compare the same price-mismatch query locally and through a fresh temporary hosted version.
Change `TEST_QUERY`, run the deployment script again, and keep the cleanup block intact.
For `PO-10234`, try these invoice prices and quantities:

| Invoiced price | Invoiced quantity | Expected percentage / total impact | Expected tier |
|---|---|---|---|
| `$14.90` | `100` | About `0.68%` / `$10` | Auto-approve |
| `$15.40` | `100` | About `4.05%` / `$60` | Supervisor sign-off, 2 business days |
| `$15.40` | `1,000` | About `4.05%` / `$600` | Procurement, 3 business days; hold payment |

Specify the **invoiced** quantity in each prompt; a partial invoice need not use the PO's full
ordered quantity. Compare facts and decisions, not exact prose, and verify deletion after every
run. Hosting the complete multi-agent workflow remains an advanced follow-on requiring separate
adapter and handoff/result validation; do not substitute it into this exercise.

## ⚠️ Troubleshooting

| Symptom | What to check |
|---|---|
| Import error for deployment models or no `create_version_from_code` | Confirm the active environment and `azure-ai-projects>=2.3.0`; compare with the current Python SDK quickstart. Do not replace source deployment with a guessed API. |
| Cannot import `ResponsesHostServer`, or its signature differs | Verify against the current Agent Framework hosting guide and installed package versions. The documented import is `from agent_framework_foundry_hosting import ResponsesHostServer`; stop and reconcile differences rather than inventing a wrapper. |
| Local server exits immediately or `/responses` is missing | Use the complete service entry point with `ResponsesHostServer(...).run()`, not Module 8's batch runner. Check port 8088 is not occupied by another process; do not terminate someone else's service. |
| Local authentication works but cloud inference returns 401/403 | Check the hosted runtime identity's model access separately from the deploying user's role. Do not upload `.azure` caches or secrets, or rely on `AzureCliCredential` inside the container. |
| Creation or invocation is denied | Verify project endpoint, tenant, Foundry Project Manager permissions, and applicable network restrictions with the facilitator. Do not bypass network or organization policies. |
| Remote build fails | Inspect build logs for Python 3.14/dependency compatibility, missing packages, local-only requirements, or unavailable package sources. Use the pinned manifest from the isolated environment and recheck current runtime support. |
| Missing CSV or wrong working directory | ZIP entries must be at the root. The tool resolves `po-records.csv` next to `main.py`, not via `LAB_REPO_ROOT`. Do not upload the entire working folder to fix one missing file. |
| Model deployment not found | `FOUNDRY_MODEL_NAME` must be the actual deployment name in this project, not a catalog model name. The entry point and deployment environment must use the same variable name. |
| Provisioning reports `"failed"` or never becomes `"active"` | Inspect the reported details and container/build logs for startup, protocol, dependencies, identity, region, or capacity problems. Let cleanup finish; do not route an unready version or treat a longer timeout as a fix. |
| Invocation reaches the wrong version | Check the dedicated agent name and saved/updated endpoint settings. This script uses the named-agent client and a 100% fixed-ratio rule, not a direct model client. Avoid concurrent routing changes. |
| Agent uses the wrong policy tier | Verify the actual lookup result and both thresholds. For the main query, `$600` exceeds the dollar limit even though `4.05%` does not exceed 5%. An HTTP success is not a correctness check. |
| Endpoint restoration fails | The nested `finally` still attempts deletion, which can leave the dedicated endpoint pointing at a deleted version. Restore its saved configuration manually and confirm both routing and deletion; never use this pattern on a shared production endpoint. |
| Deletion fails or cannot be confirmed | Treat the version as potentially billable. Inspect and delete that exact temporary version in Foundry, then verify absence. A timeout, client exit, or success message from invocation is not cleanup evidence. |
