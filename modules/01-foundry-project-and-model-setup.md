# Module 1: Foundry Project & Model Setup

**Objective:** Create a Microsoft Foundry project and deploy a chat model, understanding the
model selection and deployment options available in the portal.

**Estimated duration:** 20 minutes

**Prerequisites:** Module 0 complete — Azure sign-in verified and model quota confirmed.

## Why this module?

- **Where we are:** Module 0 established access and local tooling.
- **What's missing:** Installed tools and an Azure subscription do not yet give our application
  a model to call or a project in which to manage the agent.
- **What we'll add:** A Foundry project and a chat model deployment, chosen with capability,
  cost, and regional availability in mind. These are the foundation for later agent exercises.
- **Where it runs:** You configure resources in the browser. The project and deployed model
  live in Azure; the model does not run on your laptop.
- **By the end:** You can send a test prompt to the deployed model and identify the project
  endpoint and deployment name that later modules will use. This is still a model test, not
  yet the Invoice Discrepancy Assistant.

## Steps

### 1. Create a Foundry project

1. Browse to the [Foundry portal](https://ai.azure.com) and sign in.
2. Select **Create new** > **Project**.
3. Fill in the project details:
   - **Project name:** `foundry-practice-project` (or `<your-initials>-foundry-practice-project` if sharing a
     subscription with other workshop participants)
   - **Subscription:** `<your-subscription-name>`
   - **Resource group:** create new, e.g. `rg-foundry-practice`, or use the one your
     facilitator provided
   - **Region:** the region you confirmed has model quota in Module 0, e.g. `eastus2`
4. Select **Create** and wait for the project to finish provisioning (usually under a minute).

> **Note:** Labels and navigation may differ between versions of the New Foundry experience.
> This lab uses New Foundry throughout; the rest of these steps assume you've landed on a
> project's Home page.

### 2. Tour the project home page

1. On the project's **Home** page, locate the panel showing:
   - **Project endpoint** (you'll need this in Module 5) - shown directly with a copy button.
   - **Azure OpenAI endpoint** - a separate field, also shown directly.
   - **API key** - may show "API key authentication is disabled for this project" depending on
     your project's configuration; this lab uses Azure AD credentials (DefaultAzureCredential/
     AzureCliCredential), not API keys, so this is expected and not an error.
2. Note the two quick-start cards also on the Home page: **Build an agent** (Start building) and
   **Use a model** (View deployments) - either can take you to where you're headed next, but this
   lab uses the left-hand navigation directly.
3. Open the left-hand navigation, organized into three sections, and note where you'll return in
   later modules:
   - **Create:** Agents, Models, Services, Tools, Knowledge, Memory, Guardrails
   - **Observe:** Monitor, Traces
   - **Improve:** Data, Evaluations, Training

   Specifically note: **Models** for Module 1's deployment, **Agents** for Module 2,
   **Knowledge** for Module 3's document upload, and **Traces** for Module 10.

### 3. Browse the model catalog

1. Select **Catalog** from the left-hand navigation, then select the **Models** tab.
2. Expand **Inference tasks** in the filters and select **Chat completion**. If multiple
   similarly named options appear, select the **Chat completion** option with the largest
   model count; the count may change over time.
3. Use **`gpt-5-mini` as the suggested starting point** for this lab, if it is available with
   quota in your project region. It is a cost-conscious choice for the agent exercises and is
   supported by the Foundry IQ portal workflow used in Module 7. Otherwise, choose a current,
   non-deprecated chat model that is supported by the later modules. Avoid `gpt-4o`,
   `gpt-4o-mini`, and `gpt-4.1-nano` as defaults; check the retirement schedule before choosing
   any model. Select **Compare models** and compare candidates using these criteria:
   - **Capability:** does it reliably follow grounding/tool-use instructions?
   - **Cost:** per-token pricing, relevant since this lab runs many test queries.
   - **Context window:** large enough for policy text + PO lookups + conversation history.
   - **Region/quota availability:** must be deployable in the region you confirmed in Module 0.

> Before each delivery, check [Microsoft's model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)
> to confirm model availability and retirement dates. For example, `gpt-4.1-nano` is deprecated
> and scheduled to retire on April 14, 2027.

### 4. Deploy a chat model

1. From the model catalog, select your chosen current, non-deprecated chat model and select
   **Deploy**.
2. Configure the deployment:
   - **Deployment name:** `invoice-assistant-chat` (you'll reference this name in later modules)
   - **Deployment type:** **Global Standard** (pay-as-you-go) is sufficient for this lab; **Provisioned
     throughput** is unnecessary unless you expect sustained high load.
   - **Content filter:** leave the default filter in place for this scenario.
3. Select **Deploy** and wait for the deployment status to show **Succeeded**.

### 5. Understand deployment options

Before moving on, make sure you can explain the difference between:

- **Standard vs. Provisioned throughput:** Standard is pay-per-token. Provisioned throughput
  reserves dedicated model capacity and is billed per provisioned throughput unit (PTU) per hour,
  even when idle; it is generally for sustained workloads that need predictable throughput.
- **Global Standard, Data Zone Standard, and Standard:** these are pay-per-token options with
  different processing boundaries. Global can process requests in any Azure region. Data Zone
  keeps processing within its Microsoft-defined zone (such as US, EU, or APAC). Standard keeps
  processing within the resource's Azure geography, which can include multiple regions. The
  deployment type affects where prompts and responses are processed; it does not by itself
  determine where data is stored. Availability varies by model and region.
- **Model version and upgrade policy:** pinning a version helps make evaluation results
  reproducible, but requires you to manage upgrades before retirement. **Upgrade once new default
  version becomes available** automatically moves the deployment to the new default, which may
  change evaluation results. Record the deployed model version and policy, and review them before
  Module 10.

> Learn more about [deployment types](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
> and [model versioning](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/model-versions).

### 6. Deploy an embedding model

1. In the model catalog, select a current embedding model such as `text-embedding-3-small` and
   select **Deploy**.
2. Configure the deployment:
   - **Deployment name:** `invoice-assistant-embedding`
   - **Deployment type:** **Global Standard**, if available for the selected model and region.
3. Select **Deploy** and wait for the deployment status to show **Succeeded**. Module 7 uses this
   deployment to index and retrieve the policy in Foundry IQ.

### 7. Verify the deployments

1. In your project's left-hand navigation, select **Models**. On the **Deployments** tab, select
   **Serverless**.
2. Confirm that both the chat and embedding deployments show **Succeeded**. Note the chat
   deployment's **Name**: this is `invoice-assistant-chat` if you followed Step 4, or the name
   you chose when deploying it.
3. Select **Open playground** above the deployment list. In the **Playground** tab, check the
   **Model** selector and select your chat deployment if necessary.
4. For this simple model test, no tools are needed. If **Web search** appears under **Tools**,
   use its menu to remove it if that option is available. Check the portal notice: Web search
   can incur additional charges and send customer data outside the Azure compliance boundary.
   Use only a non-sensitive test prompt.
5. In **Chat with the model**, send a simple test prompt, for example:

   ```text
   In one sentence, what is accounts payable?
   ```

6. Confirm you get a coherent response. You do not need to select **Save as agent** here; you
   will create the agent in Module 2 using the deployment name noted above.

## ✅ Checkpoint / Validate

- [ ] Your Foundry project shows a **Succeeded** status on its Home page.
- [ ] A chat model deployment named `invoice-assistant-chat` (or your chosen name) shows status
  **Succeeded** under **Models**.
- [ ] An embedding model deployment named `invoice-assistant-embedding` (or your chosen name)
  shows status **Succeeded** under **Models**.
- [ ] You received a coherent response to a test prompt in the playground.
- [ ] You can state which model you deployed and why, using the capability/cost/context
  window/region criteria above.

## 🚀 Try it yourself

Deploy a second, smaller current, non-deprecated chat model under a different deployment name and
send it the same test prompt. Compare the response quality and latency. Keep both chat deployments
to compare them again once tools and grounding are added in later modules.

## ⚠️ Troubleshooting

- **Desired model isn't deployable in your region:** Switch to a region with available quota and
  capacity (check **Model catalog** > model page > **Regions** for availability), or pick an
  alternative model with similar capability.
- **Deployment stuck in "Creating" for several minutes:** This is usually transient. If it
  doesn't resolve within ~10 minutes, delete the deployment and retry — intermittent capacity
  issues in a region can cause this.
- **"Quota exceeded" error when deploying:** You likely still need the quota increase requested
  in Module 0. Check **Management center** > **Quota** for the current status, or deploy a model
  with lower token-per-minute requirements in the meantime.
- **Playground shows no response / times out:** Confirm the deployment status is **Succeeded**
  (not still provisioning) and that you selected the correct deployment from the playground's
  model dropdown.
