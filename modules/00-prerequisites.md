# Module 0: Prerequisites

**Objective:** Confirm you have the Azure subscription access, tooling, and local environment
needed to complete every later module, so technical setup never blocks the lab exercises.

**Estimated duration:** 15 minutes

**Prerequisites:** None — this module establishes everything later modules depend on.

## Why this module?

- **Where we are:** We have an invoice-discrepancy scenario to explore, but have not built
  an agent yet. This module establishes the starting point rather than extending a previous one.
- **What's missing:** Azure permissions, model quota, sign-in, or missing developer tools can
  block later exercises before any agent development begins.
- **What we'll add:** A checked development environment, a lab working folder, and an early
  check of cloud access and quota so setup problems can be resolved first.
- **Where it runs:** VS Code, Python, Git, and the CLI run on your computer. You use the Azure
  and Foundry portals to check cloud access; no agent application is deployed here.
- **By the end:** You can identify the subscription and region you'll use, sign in to Foundry,
  and run the local tools needed to start Module 1.

## Steps

### 1. Confirm Azure subscription access

You need an Azure subscription where you (or the workshop facilitator) can create resource
groups and Microsoft Foundry resources.

1. Sign in to the [Azure portal](https://portal.azure.com) with the account you'll use for this
   lab.
2. Confirm you can see at least one subscription:

   ```powershell
   az login
   az account show --output table
   ```

3. Confirm you have the roles needed for the full workshop. Request any missing role from your
   workshop facilitator or subscription administrator now:

   | Role | Scope | Needed for |
   |---|---|---|
   | **Contributor** | Subscription or resource group | Creating the project and chat model deployment in Module 1, plus resources in later modules, including the Module 7 embedding deployment. |
   | **Azure AI User / Foundry User** | Foundry project | Project-level data-plane access, including calling agents via SDK from Module 5 onward. |
   | **Foundry Project Manager** | Foundry project | Hosted Agent deployment in Module 9. |

   Contributor permissions alone do not provide the project-level access needed to call agents.
4. If you have access to multiple subscriptions, set the one you'll use for the lab as the
   default:

   ```powershell
   az account set --subscription "<your-subscription-name-or-id>"
   ```

### 2. Confirm Foundry portal access

1. Browse to the [Microsoft Foundry portal](https://ai.azure.com) and sign in with the same
   account.
2. Confirm the portal loads and you can see (or create) at least one project.
3. Note which **Azure region** you plan to use for this lab. Model availability and quota vary
   by region — your facilitator will tell you the recommended region for this workshop if one
   has been pre-selected (for example, `eastus2` or `swedencentral`).

### 3. Check model quota

The lab deploys a chat model in Module 1 and an embedding model in Module 7. Quota approval can
take time, so check both now.

1. In the Foundry portal, go to **Management center** > **Quota** (or the equivalent quota page
   for your portal version).
2. Check the model catalog for a current, non-deprecated chat model rather than defaulting to
   `gpt-4o` or `gpt-4o-mini`, and confirm available quota for it in your chosen region.
3. Confirm available quota for an embedding model deployment, such as `text-embedding-3-small`,
   in your chosen region, or confirm that your facilitator will provide a supported deployment.
4. If quota is needed but insufficient, submit a quota increase request now so the chat model has
   quota before Module 1 and the embedding model has quota before Module 7.

> Check [Microsoft's model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)
> before each delivery to confirm model availability and retirement dates.

### 4. Install Visual Studio Code

1. Download and install [Visual Studio Code](https://code.visualstudio.com/) if it isn't already
   installed.
2. Verify it launches and you can open the integrated terminal (`` Ctrl+` ``).

### 5. Install the Foundry Toolkit extension

1. In VS Code, open the Extensions view (`Ctrl+Shift+X`).
2. Search for **"Foundry Toolkit"** (formerly AI Toolkit) and install it.
3. Reload VS Code if prompted, and confirm a Foundry/AI Toolkit icon appears in the Activity Bar.

### 6. Install Python 3.13

1. Download and install [Python 3.13](https://www.python.org/downloads/) if it isn't already
   installed.
2. Verify the version from a terminal:

   ```powershell
   python --version
   ```

   You should see `Python 3.13.x`. If you have multiple Python versions installed, confirm the
   `python` command on your `PATH` resolves to 3.13, or use the `py -3.13` launcher on Windows.

### 7. Install Git

1. Download and install [Git](https://git-scm.com/downloads) if it isn't already installed.
2. Verify the version:

   ```powershell
   git --version
   ```

### 8. Create your lab working folder

Create a local folder you'll use for the client code written in later modules (Module 5 onward):

```powershell
New-Item -ItemType Directory -Path "$HOME\foundry-practice" -Force
Set-Location "$HOME\foundry-practice"
```

### 9. Verify end-to-end sign-in

1. Confirm `az login` (from step 1) is still signed in: `az account show`.
2. Confirm you can sign in to the [Foundry portal](https://ai.azure.com) in your browser without
   errors.

### 10. Preview the sample data you'll use later

This lab uses four sample data files — a policy document, a PO records CSV, a discrepancy-trends
CSV, and an evaluation seed file — starting in Module 3. You don't need to create or download
anything yet; see
[data/README.md](../data/README.md) for what each file will contain and which module
introduces it.

## ✅ Checkpoint / Validate

Before moving on to Module 1, confirm all of the following:

- [ ] `az account show` returns the subscription you'll use for this lab.
- [ ] You can sign in to the Foundry portal and see/create a project.
- [ ] Chat model quota is available for Module 1, and you have checked embedding quota or
  confirmed a facilitator-provided embedding deployment for Module 7.
- [ ] VS Code is installed with the Foundry Toolkit extension enabled.
- [ ] `python --version` reports Python 3.13.x.
- [ ] `git --version` returns a version number.
- [ ] Your lab working folder exists and you can `cd` into it.

## 🚀 Try it yourself

Install the [Azure CLI Bicep tools](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install)
and run `az bicep version`. You won't need Bicep for this lab, but it's a useful addition if you
plan to automate Foundry resource creation after the workshop.

## ⚠️ Troubleshooting

- **"Quota increase pending" / request not yet approved:** Quota approvals can take anywhere
  from minutes to a business day. If you're in a time-boxed live workshop and quota isn't
  approved yet, ask your facilitator about a shared/pre-provisioned project you can use instead.
- **"Access denied" creating a resource group or project:** You likely don't have Contributor
  access on the subscription. Ask your subscription administrator to grant it, or request access
  to a pre-created resource group scoped to this lab.
- **`python --version` shows an older version:** You may have multiple Python installations. On
  Windows, use `py -3.13 --version` to target 3.13 specifically, and prefix later commands with
  `py -3.13 -m ...` or adjust your `PATH`.
- **Foundry Toolkit extension not found in the Marketplace:** Some organizations restrict the VS
  Code Marketplace. Ask your IT administrator whether an offline/VSIX install is available, or
  check if your organization uses a private extension gallery.
