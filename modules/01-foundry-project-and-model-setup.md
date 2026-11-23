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
> project's overview page.

### 2. Tour the project overview

1. On the project's **Overview** page, locate:
   - The **project endpoint** (you'll need this in Module 5).
   - The list of connected resources (for example, the Azure AI Services / Cognitive Services
     resource backing your model deployments).
2. Open the left-hand navigation and note where **Models + endpoints**, **Agents**, **Files**,
   and **Tracing** live — you'll return to each of these in later modules.

### 3. Browse the model catalog

1. Select **Model catalog** from the left-hand navigation.
2. Filter by **chat completion** models.
3. Check the model catalog for a current, non-deprecated chat model rather than defaulting to
   `gpt-4o` or `gpt-4o-mini`. Compare candidates using these criteria for this scenario:
   - **Capability:** does it reliably follow grounding/tool-use instructions?
   - **Cost:** per-token pricing, relevant since this lab runs many test queries.
   - **Context window:** large enough for policy text + PO lookups + conversation history.
   - **Region/quota availability:** must be deployable in the region you confirmed in Module 0.

> Before each delivery, check [Microsoft's model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)
> to confirm model availability and retirement dates.

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

- **Global Standard (pay-as-you-go)** vs **Provisioned throughput** deployments — pay-per-token vs.
  reserved capacity for predictable, high-volume workloads.
- **Global vs. regional** deployment options, where available, and how they affect data
  residency and latency.
- **Model versioning** — new model versions are released over time; you can pin a specific
  version or take automatic updates, which matters for reproducibility in evaluation (Module 10).

### 6. Deploy an embedding model

1. In the model catalog, select a current embedding model such as `text-embedding-3-small` and
   select **Deploy**.
2. Configure the deployment:
   - **Deployment name:** `invoice-assistant-embedding`
   - **Deployment type:** **Global Standard**, if available for the selected model and region.
3. Select **Deploy** and wait for the deployment status to show **Succeeded**. Module 7 uses this
   deployment to index and retrieve the policy in Foundry IQ.

### 7. Verify the deployments

1. From the deployment's details page, select **Open in playground** (or navigate to **Playground**
   and select your chat deployment from the dropdown).
2. Send a simple test prompt, for example:

   ```text
   In one sentence, what is accounts payable?
   ```

3. Confirm you get a coherent response.
4. Note the **deployment name** (`invoice-assistant-chat`) — you'll select this model deployment
   when creating the agent in Module 2.

## ✅ Checkpoint / Validate

- [ ] Your Foundry project shows a **Succeeded** status on its overview page.
- [ ] A chat model deployment named `invoice-assistant-chat` (or your chosen name) shows status
  **Succeeded** under **Models + endpoints**.
- [ ] An embedding model deployment named `invoice-assistant-embedding` (or your chosen name)
  shows status **Succeeded** under **Models + endpoints**.
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
