# Module 1: Foundry Project & Model Setup

**Objective:** Create a Microsoft Foundry project and deploy a chat model, understanding the
model selection and deployment options available in the portal.

**Estimated duration:** 20 minutes

**Prerequisites:** Module 0 complete — Azure sign-in verified and model quota confirmed.

## Steps

### 1. Create a Foundry project

1. Browse to the [Foundry portal](https://ai.azure.com) and sign in.
2. Select **Create new** > **Project**.
3. Fill in the project details:
   - **Project name:** `invoice-discrepancy-assistant` (or `<your-initials>-invoice-assistant` if sharing a
     subscription with other workshop participants)
   - **Subscription:** `<your-subscription-name>`
   - **Resource group:** create new, e.g. `rg-invoice-discrepancy-assistant`, or use the one your
     facilitator provided
   - **Region:** the region you confirmed has model quota in Module 0, e.g. `eastus2`
4. Select **Create** and wait for the project to finish provisioning (usually under a minute).

> **Note:** Depending on your Foundry portal version, you may be prompted to create a project
> directly, or to first create (or select) a **hub** and then a project within it. Either flow
> is fine for this lab — the rest of these steps assume you've landed on a project's overview
> page.

### 2. Tour the project overview

1. On the project's **Overview** page, locate:
   - The **connection string** / project endpoint (you'll need this in Module 5).
   - The list of connected resources (for example, the Azure AI Services / Cognitive Services
     resource backing your model deployments).
2. Open the left-hand navigation and note where **Models + endpoints**, **Agents**, **Files**,
   and **Tracing** live — you'll return to each of these in later modules.

### 3. Browse the model catalog

1. Select **Model catalog** from the left-hand navigation.
2. Filter by **chat completion** models.
3. Compare two or three candidate models (for example `gpt-4o`, `gpt-4o-mini`) using these
   criteria for this scenario:
   - **Capability:** does it reliably follow grounding/tool-use instructions?
   - **Cost:** per-token pricing, relevant since this lab runs many test queries.
   - **Context window:** large enough for policy text + PO lookups + conversation history.
   - **Region/quota availability:** must be deployable in the region you confirmed in Module 0.

### 4. Deploy a chat model

1. From the model catalog, select your chosen model (for example `gpt-4o-mini`) and select
   **Deploy**.
2. Configure the deployment:
   - **Deployment name:** `invoice-assistant-chat` (you'll reference this name in later modules)
   - **Deployment type:** **Standard** (pay-as-you-go) is sufficient for this lab; **Provisioned
     throughput** is unnecessary unless you expect sustained high load.
   - **Content filter:** leave the default filter in place for this scenario.
3. Select **Deploy** and wait for the deployment status to show **Succeeded**.

### 5. Understand deployment options

Before moving on, make sure you can explain the difference between:

- **Standard (pay-as-you-go)** vs **Provisioned throughput** deployments — pay-per-token vs.
  reserved capacity for predictable, high-volume workloads.
- **Global vs. regional** deployment options, where available, and how they affect data
  residency and latency.
- **Model versioning** — new model versions are released over time; you can pin a specific
  version or take automatic updates, which matters for reproducibility in evaluation (Module 8).

### 6. Verify the deployment

1. From the deployment's details page, select **Open in playground** (or navigate to **Playground**
   and select your deployment from the dropdown).
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
- [ ] You received a coherent response to a test prompt in the playground.
- [ ] You can state which model you deployed and why, using the capability/cost/context
  window/region criteria above.

## 🚀 Try it yourself

Deploy a second, smaller model (for example `gpt-4o-mini` if you deployed `gpt-4o`, or vice
versa) under a different deployment name, and send it the same test prompt. Compare the response
quality and latency. Keep both deployments — you'll be able to compare them again once tools and
grounding are added in later modules.

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
