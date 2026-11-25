# Module 7: Foundry IQ Knowledge Base (Code)

**Objective:** Extend the **Invoice Discrepancy Assistant** from Module 3's basic File Search
grounding to an Azure AI Search-backed Foundry IQ knowledge base using Blob Storage as its
source. Retain both knowledge tools, configure approval for Foundry IQ retrieval, and handle
remote MCP approval requests in a separate Python demo.

**Estimated duration:** 45 minutes

**Prerequisites:** [Module 3](03-knowledge-base-grounding.md),
[Module 5](05-programmatic-dev-vscode-hosted-agent.md), and
[Module 6](06-mcp-integration.md) complete; permissions to create Azure AI Search and Azure
Storage resources and configure their connections; the Module 1 chat deployment (or another
supported chat completions deployment); and either a facilitator-provided embedding deployment
or permission and quota to deploy a supported embedding model in the project region (see Module 0).
The Search service needs a system-assigned managed identity and permissions to access the model
resources; assigning roles requires appropriate Azure permissions. Check your organization's
policies for Blob authentication and the separate Foundry-to-Search connection before starting.
You'll also need VS Code with the Foundry Toolkit extension. Keep your existing Python virtual
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
> in **`foundry_iq_demo.py`**. Do not edit or replace **`client.py`**: preserve it as a reference
> for its client pattern in Module 10. The demo retrieves the existing agent with `agents.get`;
> it does not create an agent version or replace its tool list. It is a policy-retrieval demo,
> not a replacement for Module 6's async PO-lookup and local MCP function dispatcher.

**Verified reference:** [Integrate an AI agent with Foundry IQ](https://microsoftlearning.github.io/mslearn-ai-agents/Instructions/Exercises/04-integrate-agent-with-foundry-iq.html)
(`mslearn-ai-agents/Instructions/Exercises/04-integrate-agent-with-foundry-iq.html`).
The reference uses product documents; here you reuse the fictional invoice/PO policy from
Module 3. Follow the portal and approval flow below rather than introducing a different
retrieval API. The portal sequence (Search service, knowledge-base form with Blob source, then
agent connection) follows [What is Foundry IQ?](https://learn.microsoft.com/azure/foundry/agents/concepts/what-is-foundry-iq?tabs=portal)
and [Connect agents to Foundry IQ knowledge bases](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect). Foundry IQ knowledge bases reached general availability in June 2026. Specific
tiers or features (for example, Serverless), regional availability, and portal labels may still
change.

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
| Approval | No MCP approval prompt | Remote MCP-style knowledge tool; requires client approval handling after you enable approval in Step 7 |
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
   This is your Module 3 baseline; keep it for Step 10. If Foundry IQ is already attached from a
   previous attempt, temporarily disconnect that knowledge tool for this baseline and reconnect
   it afterward. Do not delete the knowledge base or the File Search vector store.

### 2. Create or connect Azure AI Search from Foundry's Knowledge experience

Foundry IQ knowledge bases run on an Azure AI Search service. Create or connect that service
from the project's **Knowledge** experience so Foundry registers the connection for you:

**Order for Steps 2–5:** connect Search → enable its managed identity and check its approved
authentication settings → prepare the policy and embedding model → create the Blob source and
knowledge base → connect the knowledge base to the agent. A Search connection alone is not yet
a knowledge base.

1. In the [Foundry portal](https://ai.azure.com) (with the **New Foundry** toggle on), open your
   existing project and select **Build** from the top menu, then the **Knowledge** tab.
2. If the project already has a suitable workshop Search service connected, select it and
   continue with item 6. Do not create a duplicate service.
3. Otherwise, choose to create a new Azure AI Search service. Give it a globally unique name,
   select your workshop subscription, and use the **same resource group and region as your
   Foundry project**. The service must be in a region and tier that support agentic retrieval.
4. Choose **Basic** or an approved higher tier: this lab uses a Search managed identity for
   model access, and **Free** does not support managed identities. Basic is billable; review
   the estimated charges before creating the resource. If creation fails, use the link in the form
   to try the [Azure portal](https://portal.azure.com), then return to **Build > Knowledge** and
   connect the service you created. If the region or tier does not support agentic retrieval,
   stop and resolve availability with the facilitator rather than silently selecting a larger
   paid tier.
5. Wait for the Search service to finish creating. The **Knowledge (Foundry IQ)** page should
   now show your Search connection in the **Connection** drop-down (with a **Manage** link) and
   an enabled **Create a knowledge base** button. Don't select that button yet: finish Step 3
   first. You will use this same connection in Steps 4 and 5.
6. In a separate browser tab, open the [Azure portal](https://portal.azure.com). Search for
   **Search services**, open the Search service you selected above, and select **Settings >
   Identity**. On the **System assigned** tab, turn **Status** on and select **Save**.
   If it is already on, leave it enabled. A Free service cannot be upgraded in place; work with
   your facilitator to use a Basic or higher service instead.
7. On the Search service, select **Security + networking > Keys** and check **API Access
   control** against your organization's approved connection method:

   - For **Microsoft Entra ID**, Search must allow **Role-based access control** or **Both**.
     The user and workload identities also need the appropriate Search roles.
   - For **API Key**, Search must allow **API Key** or **Both**. Use this method only if your
     organization permits it.

   If the appropriate option is already enabled, leave it unchanged. Ask the service owner before
   changing a shared service. Do not enable API keys just to use Microsoft Entra ID.
8. Return to the Foundry **Knowledge** page and refresh it. Confirm the same connection is
   selected. Continue to Step 3 before selecting **Create a knowledge base**.

> **Same flow from the agent:** If you start from the agent's **Knowledge** page instead and
> select **Add > Connect to Foundry IQ**, a project without a Search connection shows
> **No connections available**. Select **Connect to an AI Search resource > Create new resource**
> there; it creates the same kind of project connection. Return to item 6 above to configure
> the Search identity and approved access method before continuing to Step 3.

> **Lab authentication scope:** The Foundry-to-Search connection in Step 5 can use the
> organization's approved identity or key-based configuration; the reference exercise uses a
> **Search** key. This is separate from the Blob authentication choice in Step 4. Do not paste
> keys into Python code, `.env`, screenshots, chat, or source control, or bypass a policy that
> prohibits them. Regardless of the Blob authentication choice, this lab's Search identity also
> needs access to the model resources.

### 3. Prepare the Contoso policy and embedding model

A **knowledge source** connects Foundry IQ to content. Here it points at a Blob Storage container
holding the same Contoso invoice/PO policy that Module 3's File Search uses. The source needs
two things to exist first: the uploaded policy and an embedding deployment.

> **Not creating the source yet:** In this Foundry workflow, you add the knowledge source
> *inside* the knowledge-base creation form in Step 4. First complete both preparations below.

#### Upload the policy to Blob Storage

1. In the [Azure portal](https://portal.azure.com), search for **Storage accounts**, select
   **Storage accounts**, and select **Create**. Use the workshop subscription, resource group,
   and region. Enter a globally unique name using lowercase letters and numbers, such as
   `invoicepolicyst<uniquesuffix>`. Choose **Standard** performance and **Locally-redundant
   storage (LRS)**. Review the settings and estimated cost, then create the account and wait
   for deployment to complete.
2. Open the new storage account, select **Data storage > Containers**, and create a private
   container named **`invoice-po-policy`**. Do not enable anonymous/public access.
3. Open that container, select **Upload**, and upload the exact
   [invoice-po-policy.md](../data/invoice-po-policy.md) used in Module 3.
4. Confirm the blob appears in the container. Do **not** rewrite the policy, add other documents,
   substitute the reference exercise's product documents, or remove Module 3's File Search
   upload. Both retrieval paths must use the same single policy so the later comparison is fair.

#### Deploy an embedding model

Foundry IQ uses an embedding model to turn the policy and search queries into vectors, so its
index can find relevant content by meaning. This is separate from the chat model used for
agentic retrieval.

If your facilitator provided a supported embedding deployment, confirm it is available to this
project and shows **Succeeded**, record its name, and skip the deployment steps below.

1. In the [Foundry portal](https://ai.azure.com), open your existing project. Select **Catalog**,
   then **Models**, and find `text-embedding-3-small` or another supported, current embedding
   model available in your project region.
2. Select the model and **Deploy**. Set the **Deployment name** to
   `invoice-assistant-embedding` and choose **Global Standard** if it is available and permitted
   for the selected model and region. Otherwise, choose an available deployment type approved
   for your workshop's processing boundaries.
3. Select **Deploy** and wait for the deployment to show **Succeeded** under **Models**.
   Record the actual deployment name. If quota or availability blocks deployment, check
   **Management center > Quota** and ask your facilitator for a supported option.

**Before continuing:** The policy is visible in the private Blob container, and you know the
name of a successful embedding deployment. Return to the Foundry **Knowledge (Foundry IQ)**
page with the Search connection from Step 2 selected.

### 4. Create the knowledge source and Foundry IQ knowledge base

1. On the Foundry **Knowledge (Foundry IQ)** page, select **Create a knowledge base**. If asked
   for a Search service or connection, select the existing one from Step 2; do not create another.
   The **Create a new knowledge base** page opens. Fill it in from top to bottom as follows.
2. Under **Basic configuration**, replace the generated name (for example, `knowledgebase698`)
   and add a description:

   | Field | Value for this lab |
   |---|---|
   | Name | `invoice-po-policy-kb` |
   | Description | `Contoso Finance invoice and purchase order discrepancy policy, including tolerance thresholds, duplicate detection, and missing-PO rules.` |

   Use this exact name, because Step 5 asks you to select `invoice-po-policy-kb`.
3. Expand **Improve retrieval** and select a chat completions model deployment:
   `invoice-assistant-chat` from Module 1 if listed, or another supported chat deployment.
   Set **Retrieval reasoning effort** to **Minimal** for this lab, and leave the other
   settings at their defaults. Minimal ordinarily bypasses model-based query planning; higher
   effort levels can use the chat model to plan queries. This model does not replace the
   embedding model.
4. Under **Knowledge sources (Foundry IQ)**, select **Add sources > Azure Blob Storage**. Do
   **not** select **Upload files**: that creates a different source type than the Blob
   container prepared in Step 3. In the source form, enter:

   | Field | Value for this lab |
   |---|---|
   | Name | `invoice-po-policy-source` |
   | Description | `Contoso invoice and purchase order discrepancy policy` |
   | Storage account name | The account created in Step 3 |
   | Container name | `invoice-po-policy` |
   | Authentication type | **System assigned identity** or **API Key**, according to your organization's setup (see below) |
   | Content extraction mode | **Minimal** |
   | Embedding model | `invoice-assistant-embedding`, or the supported embedding deployment you recorded |
   | Chat completions model | `invoice-assistant-chat` from Module 1 if supported, or another supported chat deployment |

   Select actual deployments, not just model catalog names. The embedding model on the source
   handles indexing and query vectors. The source form can also request a chat model, which is
   a separate setting from the **Improve retrieval** model.
5. Choose the **Authentication type** for Search to read the Blob container:

   - **System assigned identity:** Choose this when keyless access is required or preferred.
     The Search identity enabled in Step 2 needs **Storage Blob Data Reader** on the storage
     account. If the form displays **Grant access**, select it and confirm the green access
     message. If you cannot grant access, ask an administrator to assign that role to the
     Search service's system-assigned identity under the storage account's **Access control
     (IAM)**.
   - **API Key:** Choose this only if Storage account key access is enabled and permitted by
     your organization. If the option is disabled, use **System assigned identity** or ask
     your administrator; do not enable key access against policy. Follow the portal prompts
     without copying a Storage key into chat, screenshots, code, or files.

   This choice governs **Search-to-Storage** access, not the separate Search connection in
   Step 5.
6. Check the model-access message in the source form regardless of which Blob authentication
   option you selected. If prompted, select **Grant access** and confirm the green role-assigned
   message for the Search service's managed identity on the model's Foundry resource. If you
   cannot assign roles, ask an administrator for help. Microsoft documents **Cognitive Services
   User** on the Foundry resource and **Cognitive Services OpenAI User** on the resource hosting
   Azure OpenAI embedding or chat deployments. Apply the roles required by your actual model
   resources and configuration. See the [Blob knowledge source prerequisites](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-blob#prerequisites).
   Role assignments may take a few minutes to apply; if the form offers **Retry**, select it
   after propagation. Green messages confirm role assignment, not network connectivity: ensure
   the Search service can reach Storage and model resources under your organization's network
   rules.
7. Select **Create** in the source form. Back on the knowledge-base page, confirm that
   `invoice-po-policy-source` appears under **Knowledge sources**. Add only this one source so
   the comparison with File Search is fair.
8. Leave **Encryption settings** collapsed. The default Microsoft-managed keys are fine for
   this lab, unless your organization requires customer-managed keys.
9. Select **Save knowledge base**, then refresh the page until the knowledge source status
   shows **active**. If it does not become active, inspect the source/indexing errors before
   continuing. An active status alone does not confirm the policy was indexed successfully;
   verify actual retrieval in Step 6.

### 5. Connect the knowledge base to the agent

1. Return to the project's **Knowledge (Foundry IQ)** page and select **Manage** beside the
   **Connection** drop-down. This can open **Manage > Project details > Connections** (also
   labeled **Connected resources** in some portal versions). Find the row with category
   **CognitiveSearch** and confirm its target is the Search service from Step 2.
2. Check its authentication method:

   - **Microsoft Entra ID:** Keep the existing connection if this is your approved method.
     **No key available** is expected for a connection without a stored key; do not switch to
     key authentication just because that message appears. Search must accept role-based access
     as checked in Step 2.
   - **API Key:** If this is your approved method, use the existing key connection. If a
     credential must be configured, follow the portal's authentication prompts and enter the
     Search service key directly in the connection dialog. The reference exercise uses a Search
     admin key. Do not use a Storage key or share any key in chat or screenshots.

   The connection row does not by itself prove the agent can retrieve from the knowledge base:
   its MCP tool may use a separate connection or identity. If connection or retrieval fails,
   check the actual identity and permissions as described in **Troubleshooting**.
3. In Foundry, go to **Build > Agents** and open `invoice-discrepancy-assistant`. On the agent
   page, select **Playground** if it is not already selected, then find the **Knowledge**
   section in the agent configuration.
4. Select **Add > Connect to Foundry IQ**. In the dialog, choose the Search connection from
   Step 2 and the `invoice-po-policy-kb` knowledge base, then select **Connect**. If the
   knowledge base is not listed, confirm its source shows **active** in Step 4 and refresh.
5. Keep **File Search** attached as well, and save the agent. Confirm the agent's tool list now
   shows both File Search and the Foundry IQ knowledge base.
6. **Grant the agent permission to query the knowledge base.** Connecting the knowledge base
   creates a project connection named like `kb-invoice-po-policy-kb-...` that, by default,
   authenticates as the **Foundry project's managed identity**. That identity needs read access
   to Search; without it, every chat that calls Foundry IQ fails with
   **"A user error occurred while executing a tool call."**

   1. In the Azure portal, open the **Azure AI Search** service from Step 2.
   2. Select **Access control (IAM) > Add > Add role assignment**.
   3. Choose the **Search Index Data Reader** role and select **Next**.
   4. Under **Assign access to**, choose **Managed identity**, select **+ Select members**,
      set **Managed identity** to **Foundry project** (it may appear under the Foundry/AI
      Services resource type), choose your project, and select **Select**.
   5. Select **Review + assign**. Allow a few minutes for the role to take effect.

   This is the **project's** identity, not the Search service's own identity used in Step 4.
   If you cannot assign roles, ask your administrator to make this assignment. If your
   organization uses a key-based connection instead, follow its approved configuration.

### 6. Test both knowledge tools in the playground

1. In the Foundry portal, stay on the agent's **Playground** from Step 5. If you left it, go to
   **Build > Agents > invoice-discrepancy-assistant > Playground**.

   Find the **Instructions** field in the agent configuration, not the chat message box.
   In that field, find the paragraph beginning
   `You have access to Contoso's official invoice/PO discrepancy policy through file search`.
   Replace only that paragraph with the text below. Keep the other instructions, including the
   PO lookup and escalation-contact instructions, and keep the existing tools attached.

   ```text
   You have access to Contoso's official invoice/PO discrepancy policy through File Search
   and the Foundry IQ knowledge base. Use these sources to ground every threshold, escalation
   rule, or approval-limit statement. Cite the relevant policy section. When the analyst asks
   to compare retrieval tools, use the requested knowledge source and do not invent evidence
   from a source you did not retrieve. If retrieval is unavailable or denied, explain that
   limitation rather than claiming that you consulted the source.
   ```

   If you cannot find that paragraph, add the text to the **Instructions** field and remove any
   conflicting statement that the policy is available only through File Search. Select
   **Save**. Do not paste these instructions into the chat or edit `client.py`.

2. Open a fresh playground conversation and reuse these two questions from Module 3:

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

### 7. Require approval on the correct Foundry IQ tool

The Foundry portal does **not** expose this approval setting. Use the **Foundry Toolkit VS Code
extension's agent builder**:

1. In VS Code, open **Foundry Toolkit** in the left sidebar:
   - Under **My Resources**, expand the same Foundry project used in Steps 2–6 and select
     **Agents**.
   - In the **Agents** editor, select the **Prompt Agent** tab. This displays a table with
     columns such as **Name**, **Version**, and **Status**.
   - Click the blue agent name for **`invoice-discrepancy-assistant`** in that table to open
     its configuration in the agent builder. The name may be truncated; do not select a
     similarly named agent such as `InvoiceDiscrepancy...`. Open the latest version you
     just tested in the Foundry portal, rather than an older version under **Recent**.
   - Do **not** select **+ Add Prompt Agent** or **Hosted Agent**: you are editing the
     existing Prompt Agent, not creating another agent. You do not need to run or edit
     `client.py` for this step.
   - If the newly attached knowledge-base tool is missing, refresh the agent list and
     reopen the latest version. Confirm the selected project matches the portal.
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

### 8. Add a separate conversations/responses approval demo

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
        agent = project_client.agents.get(agent_name="invoice-discrepancy-assistant")
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

### 9. Run the approval and denial tests

1. Run:

   ```powershell
   python foundry_iq_demo.py
   ```

2. At the `YOU:` prompt in the VS Code terminal, enter each of these questions, one at a time:

   ```text
   What's our threshold for an automatic price-mismatch approval?
   ```

   ```text
   How do we decide if an invoice is a confirmed duplicate versus a probable duplicate?
   ```

   For **at least one**, confirm a Foundry IQ invocation produces:

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
   If it remains absent, check Step 7's tool and deployed configuration before proceeding.

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

### 10. Compare the retrieval paths and preserve your work

1. In a fresh conversation with **File Search and Foundry IQ both connected**, ask:

   ```text
   What's our threshold for an automatic price-mismatch approval?
   ```

   Do not add a source-directing prefix; this is the same question used for the Step 1 baseline.
2. Compare the baseline and new results. Record observations rather than assuming that a
   connected tool was used:

   | Observation | File Search only (Module 3 baseline) | File Search + Foundry IQ (this module) |
   |---|---|---|
   | Tool actually invoked | Record the File Search call | Record File Search, Foundry IQ, or both |
   | Policy correctness | Both thresholds and the **and** condition? | Same checks |
   | Citation detail | File name, section, excerpt, or source link shown | Record what actually appears |
   | Answer usefulness | Clear, complete, actionable? | Any improvement or regression? |
   | Interaction/latency | No MCP approval pause | Approval pauses and perceived retrieval time |

3. Include your explicitly Foundry IQ-directed run from Step 9 in the comparison if the
   unmodified question selected only File Search. Label that directed run separately so it is
   not mistaken for an identical-prompt comparison.
4. Discuss whether Foundry IQ produced more useful citation detail or a better answer. Equal
   answers are a valid result because both tools use the **same document**. Better wording alone
   is not evidence of better grounding, and Foundry IQ does not guarantee richer citations on
   every query. For a small policy pack, File Search may be sufficient and cheaper; for multiple
   enterprise knowledge sources, evaluate Foundry IQ's agentic retrieval against representative
   questions, costs, and governance requirements.
5. Leave File Search and Foundry IQ connected with the knowledge-base approval setting saved.
   Keep `foundry_iq_demo.py` separately and `client.py` unchanged as a reference for Module 10.
   Do not rerun `client.py` during this comparison: its `create_version` call uses the Modules 5–6 tool list,
   which does not include Foundry IQ. Later reuse may require reconnecting the knowledge base
   and restoring approval if you want to test Foundry IQ again.
6. Note the Search service, Storage account/container, and model deployments you added for
   cleanup in [Module 12](12-wrapup-governance.md). Resources can incur charges after this script
   exits. Remove only workshop-owned resources when finished, not shared project resources.

## ✅ Checkpoint / Validate

- [ ] Azure AI Search was created or connected from **Build > Knowledge**, is in the project's
  resource group/region, uses Basic or higher, and permits the organization's approved
  authentication method for its Foundry connection.
- [ ] The private `invoice-po-policy` Blob container holds the unchanged Module 3 policy.
- [ ] The embedding deployment you selected shows **Succeeded** and is available to the project.
- [ ] The Blob knowledge source uses the organization's approved **System assigned identity**
  or **API Key** option, **Minimal** extraction, and the selected embedding and chat deployments.
  Its status is **active**, and a retrieval test confirms the policy is accessible.
- [ ] The Search managed identity has the required model access; if it is used for Blob
  authentication, it also has access to read the storage account.
- [ ] `invoice-po-policy-kb` was connected to the agent via **Add > Connect to Foundry IQ**.
  File Search and Foundry IQ remain attached; playground evidence shows each can be invoked.
- [ ] **Never auto-approve tools** was configured on **`kb-knowledgebase...`** and saved with
  **Deploy**, not on the standalone search or Web search tool.
- [ ] `foundry_iq_demo.py` prompted for at least one Foundry IQ retrieval, accepted an approval,
  and returned a policy-grounded answer. A separate denial was also submitted and inspected.
- [ ] You can explain why zero requests is normal and how the loop handles multiple requests
  per response and repeated approval rounds.
- [ ] You compared the same question against the File Search-only baseline, checking policy
  correctness, citation detail, actual tool use, and approval overhead.
- [ ] Modules 5–6's `client.py` is unchanged as a reference for Module 10; the demo retrieves
  rather than recreates the agent. You can distinguish remote MCP approval from local MCP function dispatch.

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
- **Blob API Key is disabled or rejected:** Storage key access may be disabled by the account
  configuration or organizational policy. Choose **System assigned identity** and grant the
  Search identity Blob access, or ask the storage administrator. Do not override policy.
- **Search key authentication is rejected with a policy/compliance error:** Ask the facilitator
  for an approved identity-based configuration rather than overriding organizational policy.
- **Playground shows "A user error occurred while executing a tool call" (trace:
  `tool_user_error`, ~0 ms):** The Foundry IQ tool could not authenticate to Search. In
  **Manage > Project details > Connections**, the `kb-...` **RemoteTool** connection normally
  uses **ProjectManagedIdentity**. Grant the **Foundry project's managed identity** the
  **Search Index Data Reader** role on the Search service, as described in Step 5, wait a few
  minutes, then retry in a **new** chat.
- **Search connection or knowledge-base retrieval fails with an authentication error:** Check
  **Security + networking > Keys > API Access control** supports the selected method. For an
  API key connection, confirm the saved key belongs to that Search service and has not been
  rotated; do not use a Storage key. For Microsoft Entra ID, identify the principal used by
  the failing operation and check its Search role assignments and network access. The project
  connection shown as **CognitiveSearch** does not establish which identity the agent's MCP
  tool uses. In Microsoft's documented project-managed-identity MCP configuration, the **Foundry
  project's managed identity** needs **Search Index Data Reader** on the Search service for
  retrieval. This is distinct from the **Search service's managed identity** used for Storage
  and model access. Allow time for role propagation, and ask an administrator if roles need to
  be assigned. See [Connect agents to Foundry IQ: authentication and permissions](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect#authentication-and-permissions).
- **Blob source cannot read the document:** Confirm the selected account/container and uploaded
  blob. For **System assigned identity**, check **Storage Blob Data Reader** for the Search
  identity; for **API Key**, check that Storage key access is permitted. Allow time for role
  propagation and check that resource network rules allow the required service access. Do not
  make the container public or bypass organizational network restrictions.
- **Model access fails:** Check the role-assignment message, the Search managed identity, the
  roles on the actual model resources, and network access. Allow time for role propagation and
  use **Retry** if offered; ask an administrator if permissions cannot be assigned.
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
  Steps 5 and 7; use `foundry_iq_demo.py` for these tests.
- **Correct answer but unclear citations:** Inspect the terminal's returned annotations and
  the playground's tool/source details. Since both tools contain the same policy, mentioning
  the file name is not proof of Foundry IQ use. Record missing citation detail as a comparison
  finding rather than assuming a source or claiming an unobserved retrieval.
