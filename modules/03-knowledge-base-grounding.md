# Module 3: Knowledge Base / Grounding

**Objective:** Upload the invoice/PO policy document and ground the agent's discrepancy-handling
answers in it using File Search / Foundry IQ, understanding indexing and citations.

**Estimated duration:** 30 minutes

**Prerequisites:** Module 2 complete — the **Invoice Discrepancy Assistant** hosted Prompt Agent
exists and responds to general discrepancy questions in the playground, but cannot yet answer
policy-specific questions (for example, exact tolerance thresholds) or look up PO data.

## 1. Review the policy document

1. Open [`data/invoice-po-policy.md`](/lab/data/invoice-po-policy.md) in VS Code.
2. Skim its structure — this is Contoso Finance's (fictional) accounts-payable policy, and is the
   single source of truth the agent should ground its discrepancy-handling answers in:
   - Section 3 — **price mismatch** tolerance thresholds and escalation.
   - Section 4 — **quantity mismatch** tolerance thresholds and escalation.
   - Section 5 — **duplicate invoice** detection criteria and resolution.
   - Section 6 — **missing PO** thresholds and retroactive-PO rules.
   - Section 7 — approval authority limits by role.
   - Section 8 — an escalation contact/SLA summary table.
3. Note a few specific facts you'll use to test grounding later, for example:
   - Price mismatches ≤ 2% **and** ≤ $25 total line-item impact can be auto-approved.
   - A confirmed duplicate is an exact match on the vendor's invoice number, regardless of the
     60-day window.
   - Missing-PO invoices over $5,000 cannot get a retroactive PO and must be rejected.

> **Why a Markdown policy document?** File Search / Foundry IQ indexes text content from
> uploaded files — Markdown, with its headings and tables, chunks cleanly and keeps the tolerance
> tables intact, which makes citations easier to trace back to a specific section.

## 2. Upload the policy document to the project

1. In your Foundry project, select **Files** (or **Data** / **Knowledge**, depending on your
   portal version) from the left-hand navigation.
2. Select **+ Upload file** and choose `invoice-po-policy.md` from your local clone of this lab.
3. Wait for the upload to complete and confirm the file shows a **Ready** (or equivalent
   "processed") status before continuing — if it's still indexing, the next step may not find
   any content yet.

## 3. Attach a File Search knowledge source to the agent

1. Return to **Agents** and open the **Invoice Discrepancy Assistant** you created in Module 2.
2. In the agent's configuration, find the **Knowledge** (or **Grounding**) section and select
   **+ Add** > **File search** (shown as **Foundry IQ** in some portal versions — this lab uses
   "File search" and "Foundry IQ" interchangeably to mean the same grounding capability).
3. Create a new knowledge source / vector store and attach the `invoice-po-policy.md` file you
   just uploaded to it.
4. Save the agent configuration.

> **Indexing behavior:** uploading a file does not make it immediately searchable. Foundry chunks
> the document, generates embeddings, and builds a vector index before File Search can retrieve
> from it — for a single short Markdown file like this one, indexing typically completes within
> a minute or two, but larger files or busy regions can take longer. If grounded answers don't
> show up right away in Step 5, wait a minute and retry before troubleshooting further.

## 4. Update the system instructions

Now that grounding is attached, remove the caveat from Module 2 that told the agent it has no
knowledge base yet. Open the agent's **Instructions** field and replace the final paragraph:

```text
You do not yet have access to the policy document or PO lookup tools in this conversation unless
told otherwise — if asked a question that requires them, explain that you need that information
connected first.
```

with:

```text
You have access to Contoso's official invoice/PO discrepancy policy through file search — use it
to ground every threshold, escalation rule, or approval-limit statement you make. Cite the
relevant policy section when you reference a specific rule. You still do not have a PO lookup
tool in this conversation yet (that's added in Module 5) — if asked to look up a specific PO's
data, explain that you can't yet retrieve it.
```

Save the agent.

## 5. Re-test in the playground with policy-dependent questions

1. Open the agent's **Playground**.
2. Ask the same kind of policy question that gave a vague answer in Module 2:

   ```text
   What's our threshold for an automatic price-mismatch approval?
   ```

   Confirm the response states the **2% and $25** thresholds from Section 3.1 and includes a
   citation back to the policy document.

3. Try a few more grounding test questions that exercise each discrepancy type:

   ```text
   How do we decide if an invoice is a confirmed duplicate versus a probable duplicate?
   ```

   ```text
   An invoice came in with no matching PO and totals $6,200. What should I do?
   ```

   ```text
   A vendor over-shipped by 15 units against a 200-unit PO line. Is that within tolerance?
   ```

4. For each answer, confirm the agent surfaces a **citation** pointing back to
   `invoice-po-policy.md` (usually shown as a footnote-style marker or an expandable "Sources"
   panel, depending on your portal version) — this is how you'd let an end user verify the
   agent isn't fabricating a policy detail.

## 6. Compare against Module 2

Re-read the Module 2 playground responses to the same price-mismatch question (if you still have
that conversation open) or re-run it mentally: before grounding, the agent could only discuss the
topic generally and had no specific threshold to cite. Now it should give you the exact numbers
from Section 3.1, with a citation. This is the core value of File Search / Foundry IQ grounding —
turning general domain knowledge into policy-accurate, attributable answers.

## ✅ Checkpoint / Validate

- [ ] `invoice-po-policy.md` is uploaded to the Foundry project and shows a **Ready**/processed
  status.
- [ ] A File Search (Foundry IQ) knowledge source is attached to the **Invoice Discrepancy
  Assistant** agent and references that file.
- [ ] Asking "What's our threshold for an automatic price-mismatch approval?" returns the
  specific **2% and $25** thresholds, with a visible citation to the policy document.
- [ ] The agent correctly classifies the sample duplicate, missing-PO, and quantity-mismatch
  questions above using the policy's actual rules (not a guess).

## 🚀 Try it yourself

Ask the agent an out-of-policy-scope question, for example:

```text
What's our policy on expense report reimbursement timelines?
```

Observe how the agent handles the lack of relevant grounding — ideally it says it doesn't have
that information rather than inventing a plausible-sounding answer. If it does fabricate an
answer, revisit your system instructions from Step 4 and tighten the guidance to explicitly say
"only answer discrepancy-policy questions you can ground in the attached file search source."

## ⚠️ Troubleshooting

- **Agent still gives a vague, non-specific answer after attaching the file:** The file may still
  be indexing — check the file's status in **Files** and wait a minute before retrying. If it
  still fails after indexing completes, confirm the knowledge source in the agent configuration
  actually references this file (not an empty vector store).
- **No citation appears in the response:** Some portal playground views collapse citations into
  an expandable panel rather than inline — look for a "Sources," "References," or footnote-marker
  UI element near the response. If genuinely absent, confirm File Search is enabled for this
  specific agent (it's attached per-agent, not per-project).
- **Agent cites the document but gets a number wrong:** Re-check that you uploaded the exact file
  from this lab's `data/` folder and that it processed successfully — a partial upload or a stale
  cached index from a previous attempt can cause this. Removing and re-adding the knowledge
  source usually resolves it.
- **Upload fails or the file shows an error status:** Confirm the file is valid Markdown/plain
  text and under your portal's per-file size limit (a single-policy-document Markdown file should
  be well within any such limit).
