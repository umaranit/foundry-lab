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

Your **Azure subscription** contains a **resource group**, which groups the **Microsoft
Foundry resource** with related Azure resources. The Foundry resource hosts **Foundry projects**
and **model deployments**: a project organizes your work, while a deployment makes a named
model and version available to it. The portal can create the Foundry resource with the project;
you'll deploy the chat model separately in Step 4.

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
   - **Project endpoint** - copy and record the complete value for Module 5.
   - **Azure OpenAI endpoint** - a separate field, also shown directly.
   - **API key** - may show "API key authentication is disabled for this project" depending on
     your project's configuration; the Foundry API key shown here is not needed for this lab, so
     this is expected.

   The Foundry Project Endpoint identifies the project; the Azure OpenAI endpoint is a separate
   resource-level endpoint for Azure OpenAI model access. They are not interchangeable.
2. Note the two quick-start cards also on the Home page: **Build an agent** (Start building) and
   **Use a model** (View deployments) - either can take you to where you're headed next, but this
   lab uses the left-hand navigation directly.
3. Open the left-hand navigation, organized into three sections, and note where you'll return in
   later modules:
   - **Create:** Agents, Models, Services, Tools, Knowledge, Memory, Guardrails
   - **Observe:** Monitor, Traces
   - **Improve:** Data, Evaluations, Training

   Specifically note: **Models** for Module 1's deployment, **Agents** for Module 2,
   **Knowledge** for Module 3's document upload, **Evaluations** for Module 10, and **Traces**
   for Module 11.

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
   - **Cost:** compare input and output token prices, since this lab runs many test queries.
   - **Context window:** large enough for policy text + PO lookups + conversation history.
   - **Region/quota availability:** must be deployable in the region you confirmed in Module 0.
   - **Deployment options and processing boundaries:** check the available types and where
     requests may be processed; see Step 5 before choosing one.

> Before each delivery, check [Microsoft's model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)
> to confirm model availability and retirement dates. For example, `gpt-4.1-nano` is deprecated
> and scheduled to retire on April 14, 2027.

### 4. Deploy a chat model

1. From the model catalog, select your chosen current, non-deprecated chat model and select
   **Deploy**.
2. Configure the deployment:
   - **Deployment name:** `invoice-assistant-chat` (you'll reference this name in later modules)
   - **Deployment type:** **Global Standard** (pay-as-you-go) is sufficient for this lab if
     available and its processing boundary is permitted. Otherwise, choose an available,
     approved type; see Step 5. **Provisioned throughput** is unnecessary unless you expect
     sustained high load.
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

### 6. Verify the chat deployment

1. In your project's left-hand navigation, select **Models**. On the **Deployments** tab, select
   **Serverless**.
2. Confirm that the chat deployment shows **Succeeded**. Open its details and record:
   - The **Foundry Project Endpoint** copied in Step 2.
   - The **chat model deployment name**: `invoice-assistant-chat`, or the name you chose.
   - The **model name and version** shown in the deployment details.
   - The **deployment type** you selected.
   - The **project/resource region** shown in the project or resource details.
3. Return to the deployment list and select **Open playground**. In the **Playground** tab,
   check the **Model** selector and select your chat deployment if necessary.
4. For this model-only test, do not attach the policy or add tools. If **Web search** is enabled
   under **Tools**, remove or disable it; ask your facilitator if neither option is available.
   Check the portal notice: Web search can incur additional charges and send customer data outside
   the Azure compliance boundary. Use only a non-sensitive test prompt.
5. In **Chat with the model**, send this test prompt:

   ```text
   How would you identify an invoice discrepancy based on Contoso's invoice policies?
   ```

6. Confirm you get a coherent response. The model can explain a general approach, but it does not
   yet have access to Contoso's specific invoice policies; do not treat any policy details it
   guesses as verified. You do not need to select **Save as agent** here.

In [Module 2](02-build-the-agent-portal.md), you'll give this model an agent role and instructions;
[Module 3](03-knowledge-base-grounding.md) then adds the actual policy as grounding.

## ✅ Checkpoint / Validate

- [ ] Your Foundry project shows a **Succeeded** status on its Home page.
- [ ] A chat model deployment named `invoice-assistant-chat` (or your chosen name) shows status
  **Succeeded** under **Models**.
- [ ] You recorded the Foundry Project Endpoint, chat deployment name, model name and version,
  deployment type, and project/resource region.
- [ ] The test prompt produced a coherent general response; you understand that the model has
  not yet accessed Contoso's specific policies.
- [ ] You can explain your choice using capability, input/output token pricing, context window,
  regional availability/quota, and deployment processing boundaries.

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
