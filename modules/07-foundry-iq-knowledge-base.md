# Module 7: Foundry IQ Knowledge Base (Code)

**Objective:** Extend the **Invoice Discrepancy Assistant** from Module 3's basic File Search
grounding to an Azure AI Search-backed Foundry IQ knowledge base using Blob Storage as its
source. Retain both knowledge tools, configure approval for Foundry IQ retrieval, and handle
remote MCP approval requests in a separate Python demo.

**Estimated duration:** 45 minutes

**Prerequisites:** [Module 3](03-knowledge-base-grounding.md),
[Module 5](05-programmatic-dev-vscode-hosted-agent.md), and
[Module 6](06-mcp-integration.md) complete; permissions to create Azure AI Search and Azure
Storage resources and configure their connections; an available embedding model and chat
completions model; VS Code with the Foundry Toolkit extension. Keep your existing Python virtual
environment, `.env`, File Search vector store, and Module 6 client.

## Why this module?

- **Where we are:** Our Prompt Agent has basic File Search grounding and custom tools,
  including a local MCP integration from Module 6.
- **What's missing:** We have not explored a multi-source-capable knowledge base or the
  approval flow for a remote MCP-style retrieval tool. Local stdio MCP does not teach that flow.
- **What we'll add:** A Foundry IQ knowledge base alongside File Search, using the same policy
  for comparison, and a separate demo that handles explicitly enabled retrieval approvals.
  This explores another retrieval option; it does not assume File Search is inadequate.
- **Where it runs:** Azure AI Search and Blob Storage support the cloud knowledge base.
  Your local demo submits requests and approval decisions; it does not run the retrieval
  server as a subprocess. The Module 6 client remains unchanged.
- **By the end:** You can compare retrieval evidence and citations, approve or deny a Foundry IQ
  request, and explain the additional setup and cost without assuming it improves every answer.

> **Sequence note:** This module follows MCP Integration and builds on Modules 5–6's
> conversations/responses client pattern, but puts the new knowledge-and-approval demonstration
> in **`foundry_iq_demo.py`**. Do not edit or replace **`client.py`**: preserve it for reuse in
> Module 9. The demo retrieves the existing agent with `agents.get`; it does not create an agent
> version or replace its tool list. It is a policy-retrieval demo, not a replacement for Module
> 6's async PO-lookup and local MCP function dispatcher.

**Verified reference:** [Integrate an AI agent with Foundry IQ](https://microsoftlearning.github.io/mslearn-ai-agents/Instructions/Exercises/04-integrate-agent-with-foundry-iq.html)
(`mslearn-ai-agents/Instructions/Exercises/04-integrate-agent-with-foundry-iq.html`).
The reference uses product documents; here you reuse the fictional invoice/PO policy from
Module 3. Follow the portal and approval flow below rather than introducing a different
retrieval API. The reference's preview/active-development caveat applies: portal labels,
regional availability, and supported model choices can change.

## Steps

### 1. Contrast File Search with Foundry IQ

File Search and Foundry IQ are **distinct retrieval options**, not synonyms. **File Search**
means the single vector store you
attached there, and **Foundry IQ** means the Azure AI Search-backed knowledge base you create now.

| Aspect | Module 3: File Search | Module 7: Foundry IQ |
|---|---|---|
| Knowledge scope | A single vector store containing the uploaded policy file in this lab | Multi-source-capable knowledge base; this lab starts with one Blob Storage container |
| Retrieval | Simpler file retrieval with citations | Agentic retrieval that can plan searches across connected knowledge sources |
| Resources | No separately provisioned Azure AI Search resource in this lab | Azure AI Search, Blob Storage, an embedding model, and a chat completions model |
| Approval | No MCP approval prompt | Remote MCP-style knowledge tool; requires client approval handling after you enable approval in Step 6 |
| Complexity and cost | Simpler and generally cheaper for this small, single-policy scenario; not necessarily free | More setup and potential Search, Storage, embedding, and retrieval-model charges |
| When to choose | A small, stable set of files and straightforward grounding | Knowledge spanning multiple sources, agentic retrieval, and an explicit retrieval-approval workflow |

Foundry IQ is not automatically more accurate for one short document. Its broader retrieval
capabilities become useful as an engagement's knowledge sources grow. Compare actual answers
and citations before accepting the extra cost and operational complexity.

1. Before adding Foundry IQ, open the existing agent's playground in a **new conversation**.
2. With **only File Search available as the policy-retrieval tool**, ask:

   ```text
   What's our threshold for an automatic price-mismatch approval?
   ```

3. Record the answer, visible citation/source detail, and the File Search tool-call evidence.
   This is your Module 3 baseline; keep it for Step 9. If Foundry IQ is already attached from a
   previous attempt, temporarily disconnect that knowledge tool for this baseline and reconnect
   it afterward. Do not delete the knowledge base or the File Search vector store.

### 2. Create Azure AI Search and enable key access

1. In the [Azure portal](https://portal.azure.com), select **Create a resource**, search for
   **Azure AI Search**, and create a search service.
2. Select your workshop subscription and the **same resource group and region as your Foundry
   project**. Give the service a unique name, such as `invoice-policy-search-<unique-suffix>`.
3. Use the default/**Free** tier where available for the exercise, or **Basic** if Free is not
   available. Review the selected tier and estimated charges before creating the resource;
   Basic is billable. If your region/tier does not support the required capability, stop and
   resolve availability with the facilitator rather than silently selecting a larger paid tier.
4. Once deployment completes, open the Search resource and go to **Security + networking >
   Keys**.
5. Set **API Access control** to **Both** and save. This allows the key-authentication path used
   by the reference exercise alongside role-based access.
6. Keep this page available: you will use the Search resource's key when configuring the
   Foundry connection in Step 4.

> **Lab authentication scope:** API keys are used here to mirror the verified reference, not as
> a blanket production recommendation. Do not paste keys into Python code, `.env`, screenshots,
> chat, or source control. Enter the key only into the connection's credential field. Follow
> your organization's identity, key-management, and network policies for a real deployment; do
> not bypass a policy that prohibits key authentication.

### 3. Create Blob Storage and upload the existing policy

1. In the Azure portal, create a **Storage account** in the workshop resource group and region.
   Use a globally unique name with lowercase letters and numbers, such as
   `invoicepolicyst<uniquesuffix>`. Standard storage with locally redundant storage is sufficient
   for this lab; review its estimated cost before creating it.
2. Open the storage account, select **Data storage > Containers**, and create a private
   container named **`invoice-po-policy`**. Do not enable anonymous/public access.
3. Open that container, select **Upload**, and upload the exact
   [invoice-po-policy.md](../data/invoice-po-policy.md) used in Module 3.
4. Confirm the blob appears in the container. Do **not** rewrite the policy, substitute the
   reference exercise's product documents, or remove Module 3's File Search upload. Both
   retrieval paths must use the same policy so the later comparison is meaningful.

### 4. Create and connect the Foundry IQ knowledge base

1. Open your project in the [Foundry portal](https://ai.azure.com), then open the
   **Invoice Discrepancy Assistant** agent and its **Knowledge** page.
2. Select **Create a knowledge base**. Use the Azure AI Search resource you created in Step 2
   when selecting the search connection.
3. Add a source of type **Azure Blob Storage** and configure the knowledge base/source form:

   | Field | Value for this lab |
   |---|---|
   | Name | `invoice-po-policy-kb` for the knowledge base; `invoice-po-policy-source` if a separate source name is requested |
   | Description | `Contoso Finance invoice and purchase order discrepancy policy, including tolerance thresholds, duplicate detection, and missing-PO rules.` |
   | Storage account | The account created in Step 3 |
   | Blob container | `invoice-po-policy` |
   | Authentication type | **API Key** |
   | Content extraction mode | **minimal** |
   | Embedding model | An available embedding model deployment in your project |
   | Chat completions model | An available, supported chat completions model deployment in your project |

   Select actual deployments, not just model catalog names. The embedding model supports
   indexing/retrieval; the chat completions model supports agentic retrieval. These are separate
   roles even if you reuse the agent's chat deployment for the latter.

4. Save and wait until the knowledge base/source status is **active**. Saving configuration is
   not proof that the policy is indexed and ready.
5. Return to the agent's **Knowledge** page and open **Connection > Manage**.
6. Under **Connected resources**, find the Search service from Step 2. Set **Authentication** to
   **Key authentication**, paste that **Search resource's key** from its **Keys** page, and save.
   This is a separate setting from the Blob source's **API Key** authentication; do not paste a
   Storage account key into the Search connection.
7. Return to the agent and confirm the knowledge base is attached. If the creation flow has
   not attached it, use **Add > Connect to Foundry IQ** and select the search connection and
   `invoice-po-policy-kb`. Keep **File Search** attached as well, and save the agent.

### 5. Test both knowledge tools in the playground

1. Keep the existing instructions and tools from Modules 5–6. Update only the grounding
   paragraph that currently says the policy is available through file search:

   ```text
   You have access to Contoso's official invoice/PO discrepancy policy through File Search
   and the Foundry IQ knowledge base. Use these sources to ground every threshold, escalation
   rule, or approval-limit statement. Cite the relevant policy section. When the analyst asks
   to compare retrieval tools, use the requested knowledge source and do not invent evidence
   from a source you did not retrieve. If retrieval is unavailable or denied, explain that
   limitation rather than claiming that you consulted the source.
   ```

2. Save, open a fresh playground conversation, and reuse these two questions from Module 3:

   ```text
   What's our threshold for an automatic price-mismatch approval?
   ```

   ```text
   How do we decide if an invoice is a confirmed duplicate versus a probable duplicate?
   ```

3. Inspect the tool-call details and sources, not just the answer text. For a diagnostic retry,
   prepend **"Use File Search to answer this policy question:"** to one question and
   **"Use the Foundry IQ knowledge base to answer this policy question:"** to the other.
   Confirm you can observe **both** tools being invoked across these tests; they need not both
   run in every turn.
4. Check the answer against the same source document:
   - **Price mismatch:** Section 3.1 requires both **≤ 2%** and **≤ $25 total line-item impact**.
     The answer must not replace **and** with **or**.
   - **Duplicates:** Section 5.1 distinguishes an exact vendor invoice-number match, regardless
     of the 60-day window (subject to the written-reuse exception), from the probable-duplicate
     criteria: same vendor, matching PO or the specified no-PO alternative, total within
     **$1.00**, and submission within **60 calendar days**.

> **No approval prompt yet is expected.** The portal-created Foundry IQ tool runs without
> approval by default. You explicitly change that next; approval is not an inherent requirement
> of every Foundry IQ deployment.

### 6. Require approval on the correct Foundry IQ tool

The Foundry portal does **not** expose this approval setting. Use the **Foundry Toolkit VS Code
extension's agent builder**:

1. In VS Code, open Foundry Toolkit, select the same project, and open the existing
   **Invoice Discrepancy Assistant** in the agent builder. Refresh if the new knowledge tool
   does not appear.
2. In the agent's tool list, find the tool whose generated name starts with
   **`kb-knowledgebase...`**. Confirm it corresponds to your Foundry IQ knowledge base.
3. Select that tool's **Configure** action and choose **Never auto-approve tools**.
4. Select **Deploy** to save the updated agent configuration. Merely changing the local builder
   setting is not sufficient.
5. Verify the saved agent still has File Search and the other tools from Modules 5–6, alongside
   the knowledge-base tool.

> **Choose the right tool:** The builder may also show a default **Web search** tool and a
> standalone **Azure AI Search** tool. Neither is the Foundry IQ `kb-knowledgebase...` tool.
> Configuring approval on either of those instead will not produce the approval flow this lab
> is testing. Do not remove unrelated tools or change their approval settings for this exercise.

### 7. Add a separate conversations/responses approval demo

1. Return to the Module 0 working folder and activate the same virtual environment:

   ```powershell
   Set-Location "$HOME\foundry-practice"
   .\.venv\Scripts\Activate.ps1
   ```

2. Reuse the `PROJECT_ENDPOINT` value in the existing `.env` and your Azure sign-in from
   Module 5. The already-installed `azure-ai-projects`, `azure-identity`, `openai`, and
   `python-dotenv` packages are sufficient for this demo.
3. Create **`foundry_iq_demo.py`** beside `client.py`, with the complete code below. Do not import
   `client.py` or copy its `create_version` call: recreating the agent from the old tool list can
   omit the newly attached knowledge base and approval configuration.

```python
import os

from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv

load_dotenv()


def main():
    with (
        DefaultAzureCredential() as credential,
        AIProjectClient(
            endpoint=os.environ["PROJECT_ENDPOINT"], credential=credential
        ) as project_client,
        project_client.get_openai_client() as openai_client,
    ):
        agent = project_client.agents.get(agent_name="Invoice Discrepancy Assistant")
        conversation = openai_client.conversations.create(items=[])
        print("Foundry IQ policy demo ready. Type 'exit' to quit.\n")

        while True:
            user_message = input("YOU: ").strip()
            if user_message.lower() in {"exit", "quit"}:
                break
            if not user_message:
                print("Enter a policy question, or type 'exit'.")
                continue

            openai_client.conversations.items.create(
                conversation_id=conversation.id,
                items=[{"type": "message", "role": "user", "content": user_message}],
            )
            response = openai_client.responses.create(
                conversation=conversation.id,
                extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
                input="",
            )

            while True:
                approval_requests = [
                    item for item in (getattr(response, "output", None) or [])
                    if getattr(item, "type", None) == "mcp_approval_request"
                ]
                if not approval_requests:
                    break

                approval_items = []
                for approval_request in approval_requests:
                    print(f"[Approval required for: {approval_request.name}]")
                    print(f"Server: {approval_request.server_label}")
                    approved = input("Approve this action? (yes/no): ").strip().lower() in ("yes", "y")
                    approval_items.append({
                        "type": "mcp_approval_response",
                        "approval_request_id": approval_request.id,
                        "approve": approved,
                    })

                openai_client.conversations.items.create(
                    conversation_id=conversation.id, items=approval_items,
                )
                response = openai_client.responses.create(
                    conversation=conversation.id,
                    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
                    input="",
                )

            output_items = getattr(response, "output", None) or []
            if any(getattr(item, "type", None) == "function_call" for item in output_items):
                raise RuntimeError(
                    "This policy-only demo does not dispatch local function tools. "
                    "Restart it and ask a policy-retrieval question without a PO lookup "
                    "or escalation-contact lookup. Module 6's client retains that dispatcher."
                )
            if response.error is not None:
                raise RuntimeError(f"Agent response failed: {response.error}")
            if not response.output_text:
                raise RuntimeError(
                    f"No final answer text was returned (status: {response.status}). "
                    "Inspect the response and tool execution before continuing."
                )

            print(f"AGENT: {response.output_text}\n")
            for item in output_items:
                print(f"[Response item: {item.type}]")
                if item.type == "message":
                    for content in item.content:
                        if content.type == "output_text":
                            for annotation in content.annotations:
                                print(f"[Citation: {annotation.model_dump_json()}]")


if __name__ == "__main__":
    main()
```

**How the approval loop works**

- **Zero requests is valid.** The agent does not always request approval. It may answer through
  File Search, use existing conversation context, or not need a tool. An empty
  `approval_requests` list exits the inner loop immediately.
- **Several requests can arrive in one turn.** The `for` loop asks for a decision on every
  `mcp_approval_request` in that response, then submits all decisions together. Each
  `approval_request_id` must match the request's `id`; the tool name is not that identifier.
- **Approval can take multiple rounds.** After adding the decisions to the same conversation,
  the client calls `responses.create` again with the same agent reference and `input=""`.
  The new response can contain more requests, so the loop continues until
  `approval_requests` is empty. A single `if` statement or one approval per user message would
  miss these cases.
- **Denial is a real decision.** Only `yes` or `y` approves. `no` and all other entries submit
  `approve=False`; the client still resumes the response so the agent can explain the
  limitation or use another available source.

This approval loop mirrors the verified reference exactly. The surrounding input/output and
error checks make it a standalone lab script. Citation annotations are printed when returned;
their absence is not proof of a particular tool being used. Inspect playground tool details
as well.

Unlike Module 6, this script does not launch a local FastMCP process or use
`stdio_client` / `ClientSession`. The Foundry IQ knowledge tool is remote, and Foundry executes
the approved retrieval. Module 6's local MCP tool is wrapped as a `FunctionTool` and executed
by its async client, without this remote approval loop. Keep the two client responsibilities
separate for this exercise; the existing agent's local function definitions remain attached,
but this focused demo deliberately does not dispatch them.

### 8. Run the approval and denial tests

1. Run:

   ```powershell
   python foundry_iq_demo.py
   ```

2. Ask the same two Module 3 policy questions from Step 5. For **at least one**, confirm a
   Foundry IQ invocation produces:

   ```text
   [Approval required for: <requested-tool-name>]
   Server: <knowledge-base-server-label>
   Approve this action? (yes/no):
   ```

   Check that the name/server correspond to the knowledge-base tool you configured, not Web
   search or a standalone search tool. Enter **yes**, then verify a grounded answer with the
   correct policy details. If further prompts appear, review each request before answering.

3. If neither question triggers approval, exit and restart for a fresh conversation, then ask:

   ```text
   Use the Foundry IQ knowledge base to answer this policy question, not File Search:
   What's our threshold for an automatic price-mismatch approval?
   ```

   A correct answer alone is not a pass: you must observe the knowledge-base approval prompt.
   If it remains absent, check Step 6's tool and deployed configuration before proceeding.

4. Exit and restart again. Ask the explicitly Foundry IQ-directed question and enter **no**.
   Confirm the denied retrieval is not reported as successfully consulted. File Search remains
   available, so denial does not necessarily mean there will be no answer; inspect the evidence
   for any fallback and ensure it is not misattributed to Foundry IQ.
5. If the agent requests approval again after denial, decide on that new request independently;
   the loop must not automatically approve it. Type `exit` at the next `YOU:` prompt when done
   (or press `Ctrl+C` to stop during an approval sequence).

Do not force the model to emit multiple requests just to pass the lab. If a response contains
several requests or a later response asks again, confirm the client prompts for each. The loop
handles zero, one, multiple, and repeated requests regardless of which pattern your run produces.

### 9. Compare the retrieval paths and preserve your work

1. In a fresh conversation with **File Search and Foundry IQ both connected**, ask the exact
   price-mismatch question you saved in Step 1, without a source-directing prefix.
2. Compare the baseline and new results. Record observations rather than assuming that a
   connected tool was used:

   | Observation | File Search only (Module 3 baseline) | File Search + Foundry IQ (this module) |
   |---|---|---|
   | Tool actually invoked | Record the File Search call | Record File Search, Foundry IQ, or both |
   | Policy correctness | Both thresholds and the **and** condition? | Same checks |
   | Citation detail | File name, section, excerpt, or source link shown | Record what actually appears |
   | Answer usefulness | Clear, complete, actionable? | Any improvement or regression? |
   | Interaction/latency | No MCP approval pause | Approval pauses and perceived retrieval time |

3. Include your explicitly Foundry IQ-directed run from Step 8 in the comparison if the
   unmodified question selected only File Search. Label that directed run separately so it is
   not mistaken for an identical-prompt comparison.
4. Discuss whether Foundry IQ produced more useful citation detail or a better answer. Equal
   answers are a valid result because both tools use the **same document**. Better wording alone
   is not evidence of better grounding, and Foundry IQ does not guarantee richer citations on
   every query. For a small policy pack, File Search may be sufficient and cheaper; for multiple
   enterprise knowledge sources, evaluate Foundry IQ's agentic retrieval against representative
   questions, costs, and governance requirements.
5. Leave File Search and Foundry IQ connected with the knowledge-base approval setting saved.
   Keep `foundry_iq_demo.py` separately and `client.py` unchanged for Module 9. Do not rerun
   `client.py` during this comparison: its `create_version` call uses the Modules 5–6 tool list,
   which does not include Foundry IQ. Later reuse may require reconnecting the knowledge base
   and restoring approval if you want to test Foundry IQ again.
6. Note the Search service, Storage account/container, and model deployments you added for
   cleanup in [Module 12](12-wrapup-governance.md). Resources can incur charges after this script
   exits. Remove only workshop-owned resources when finished, not shared project resources.

## ✅ Checkpoint / Validate

- [ ] Azure AI Search is in the project's resource group/region, with **API Access control =
  Both**; its Foundry connection uses **Key authentication**.
- [ ] The private `invoice-po-policy` Blob container holds the unchanged Module 3 policy.
- [ ] The Blob knowledge source uses **API Key**, **minimal** extraction, an embedding model,
  and a chat completions model, and its status is **active**.
- [ ] File Search and Foundry IQ remain attached; playground evidence shows each can be invoked.
- [ ] **Never auto-approve tools** was configured on **`kb-knowledgebase...`** and saved with
  **Deploy**, not on the standalone search or Web search tool.
- [ ] `foundry_iq_demo.py` prompted for at least one Foundry IQ retrieval, accepted an approval,
  and returned a policy-grounded answer. A separate denial was also submitted and inspected.
- [ ] You can explain why zero requests is normal and how the loop handles multiple requests
  per response and repeated approval rounds.
- [ ] You compared the same question against the File Search-only baseline, checking policy
  correctness, citation detail, actual tool use, and approval overhead.
- [ ] Modules 5–6's `client.py` is unchanged for Module 9; the demo retrieves rather than
  recreates the agent. You can distinguish remote MCP approval from local MCP function dispatch.

## 🚀 Try it yourself

Reuse another Module 3 question:

```text
A vendor over-shipped by 15 units against a 200-unit PO line. Is that within tolerance?
```

Ask for policy interpretation only, first through File Search and then through Foundry IQ in
fresh conversations. Check that the answer computes **7.5%**, recognizes that both the **3%**
and **10-unit** thresholds are exceeded, cites Section 4.1, and does not assume the receiving
report confirms all goods. Approve the Foundry IQ request only after reviewing the tool/server.
Compare how clearly each answer distinguishes a policy rule from an actual PO/receiving lookup.

## ⚠️ Troubleshooting

- **Search creation or model selection is blocked:** Check resource-creation permissions,
  region, tier availability, and model deployment/quota. A chat model is not an embedding model.
  Use the facilitator's supported setup rather than making an unapproved tier or region change.
- **Key-based authentication is rejected with a policy/compliance error:** Unlike a typo or
  wrong key, this likely indicates an organizational Azure Policy restriction on Search or
  Storage, not a lab mistake. Escalate to the subscription owner rather than retrying the same
  steps.
- **Search connection fails with an authentication error:** Check **Security + networking >
  Keys > API Access control = Both**, then **Knowledge > Connection > Manage > Connected
  resources > Authentication = Key authentication**. Confirm the saved key belongs to that
  Search service and has not been rotated; do not use the Storage key in this field.
- **Blob source cannot read the document:** Confirm the selected account/container and
  **API Key** authentication. Check that the blob uploaded successfully and that resource
  network rules allow the required service access. Do not make the container public or bypass
  organizational network restrictions to fix an access error.
- **Knowledge base remains inactive or returns no policy content:** Check source/indexing
  status and error details, the **minimal** extraction setting, and both selected model
  deployments. Wait for **active** before retrying; recreating the agent will not fix indexing.
- **No approval prompt:** First check actual tool use. File Search answering is valid and does
  not need approval. Retry in a fresh conversation with the Foundry IQ-directed question. If
  Foundry IQ runs without prompting, reopen the correct **`kb-knowledgebase...`** tool in the
  builder, select **Never auto-approve tools**, and **Deploy**. Check that VS Code, the portal,
  and `PROJECT_ENDPOINT` all target the same project and agent.
- **Approval configured, but only Web search or standalone search prompts:** You changed the
  wrong tool. Configure the knowledge-base tool specifically; those other tools are not
  substitutes for it.
- **Client stops after one approval or a later tool request stalls:** Retain the entire inner
  `while True` loop. Submit every request's matching `approval_request_id` to the same
  conversation, then call `responses.create` again with the agent reference and `input=""`.
- **A denial is followed by an answer:** File Search or prior conversation context may still be
  available. Start fresh for the denial test and inspect sources; the agent must not claim the
  denied Foundry IQ call succeeded.
- **Demo reports a local function call:** The request selected a PO lookup or escalation-contact
  function, which this focused script does not execute. Restart and ask one of the policy-only
  questions above. Keep the Module 6 client for its original local-tool workflow; do not
  overwrite it with this demo or rerun its version-creation code as an approval fix.
- **Foundry IQ disappears after running the older client:** Its `create_version` tool list
  predates this module. Reconnect the knowledge base and redeploy its approval setting using
  Steps 4 and 6; use `foundry_iq_demo.py` for these tests.
- **Correct answer but unclear citations:** Inspect the terminal's returned annotations and
  the playground's tool/source details. Since both tools contain the same policy, mentioning
  the file name is not proof of Foundry IQ use. Record missing citation detail as a comparison
  finding rather than assuming a source or claiming an unobserved retrieval.
