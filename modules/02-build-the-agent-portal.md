# Module 2: Build the Agent (Portal)

**Objective:** Create the Invoice Discrepancy Assistant as a Prompt Agent in the Foundry
portal, write its system instructions, and test it in the playground.

This is a Prompt Agent (AgentKind.PROMPT) - server-managed by Foundry, not to be confused with the separate Hosted Agent kind (AgentKind.HOSTED) you'll meet in Module 9.

**Estimated duration:** 25 minutes

**Prerequisites:** Module 1 complete — a Foundry project exists with the
`invoice-assistant-chat` model deployment showing status **Succeeded**.

## Steps

### 1. Navigate to Agents

1. In your Foundry project, select **Agents** from the left-hand navigation.
2. Select **+ New agent**.

### 2. Create the Prompt Agent

1. Choose the **Prompt agent** type (a server-hosted agent configured declaratively in the
   portal — as opposed to the code-first agents you'll build in Module 8).
2. Configure the basics:
   - **Name:** `Invoice Discrepancy Assistant`
   - **Model deployment:** `invoice-assistant-chat` (from Module 1)
3. Select **Create**.

### 3. Write the system instructions

Open the agent's **Instructions** field and enter something like the following. Adjust wording
to your own style, but keep the role, scope, and guardrails:

```text
You are the Invoice Discrepancy Assistant for Contoso Finance's accounts payable team.

Your role is to help AP analysts triage invoice discrepancies, specifically:
- Price mismatches (invoiced unit price differs from the purchase order)
- Quantity mismatches (invoiced quantity differs from the purchase order)
- Duplicate invoices
- Missing purchase orders

Guidelines:
- Always ground discrepancy-handling guidance (thresholds, escalation steps, approval rules) in
  Contoso's official policy — never invent a threshold, rule, or policy detail.
- Never invent or guess purchase order data (PO number, vendor, price, quantity, status). Only
  state PO details you can retrieve from an authoritative source.
- If you don't have enough information to answer confidently, say so and explain what
  information is missing, rather than guessing.
- Keep responses concise and action-oriented for a busy AP analyst.
- When you cite a policy rule, make it clear you're citing company policy.

You do not yet have access to the policy document or PO lookup tools in this conversation unless
told otherwise — if asked a question that requires them, explain that you need that information
connected first.
```

> **Why mention missing tools/grounding?** At this point in the lab, the agent has no knowledge
> base or tools attached yet (those start coming in Module 3). Calling this out in the
> instructions keeps the agent from confidently guessing policy details or PO data it doesn't
> actually have — you'll remove this caveat once grounding and tools are in place.

### 4. Note how this agent differs from Module 8

This agent is a **Prompt Agent**: its definition (instructions, model, and — starting in
Modules 3–7 — knowledge and tools) lives in the Foundry service and is configured here in the
portal (and, from Module 5 onward, extended via code as new agent versions). In Module 8, you'll
build a **separate, code-first** multi-agent system with the Microsoft Agent Framework SDK. Keep
this distinction in mind — Modules 2–7 are all about this one hosted agent; Module 8 is a
different, code-defined system that happens to address the same scenario.

### 5. Test in the playground

1. Select **Try in playground** (or open the **Playground** tab for this agent).
2. Try a few baseline questions, for example:

   ```text
   What should I do if an invoice's unit price doesn't match the purchase order?
   ```

   ```text
   Look up PO number PO-10234 for me.
   ```

   ```text
   What's our policy on duplicate invoices?
   ```

3. Observe that the agent can discuss the topic generally but:
   - Cannot give you a specific policy threshold (no grounding yet).
   - Cannot actually look up PO-10234 (no tool yet) — ideally it says so rather than fabricating
     an answer, per the instructions above.

### 6. Save the agent

1. Confirm your instructions are saved (most portal versions autosave, but explicitly select
   **Save** if available).
2. Note the agent's name/ID — you'll reference it again in Module 3 (adding grounding), Module 4
   (adding tools), and Module 5 (calling it from code).

## ✅ Checkpoint / Validate

- [ ] The **Invoice Discrepancy Assistant** agent exists and uses the `invoice-assistant-chat`
  deployment.
- [ ] The agent responds sensibly and on-topic to a general discrepancy question in the
  playground.
- [ ] When asked to look up a specific PO number, the agent indicates it cannot do so yet, rather
  than inventing a plausible-sounding answer.
- [ ] You can explain, in one or two sentences, why this agent can't yet answer policy-specific
  questions accurately (no knowledge base attached).

## 🚀 Try it yourself

Edit the system instructions to add a stricter escalation tone (for example, "if the discrepancy
exceeds $500, state that it must be escalated to a supervisor regardless of other factors") and
re-test the same three playground prompts. Compare how the tone and content of the responses
change before reverting the hard-coded threshold — explaining why hard-coding policy values in
instructions, rather than grounding them in a document, becomes a maintenance problem is a good
lead-in to Module 3.

## ⚠️ Troubleshooting

- **Model deployment doesn't appear in the dropdown when creating the agent:** Confirm the
  Module 1 deployment status is **Succeeded** and that you're creating the agent in the same
  project where you deployed the model.
- **Agent gives a confident-sounding but made-up PO lookup result:** This is expected at this
  stage since no tool is attached yet — it's a good discussion point for why grounding (Module 3)
  and tools (Module 4) matter, but you may want to tighten the instructions further if it
  happens too often even with the caveat included above.
- **Instructions field rejects your text / truncates it:** Some portal versions cap system
  instruction length. If you hit a limit, trim guidance that will be covered by the policy
  document in Module 3 instead of duplicating it in the instructions.
- **Playground shows an error instead of a response:** Confirm the underlying model deployment
  still shows **Succeeded** under **Models + endpoints** — a deployment can occasionally be
  deleted or fail after creation.
