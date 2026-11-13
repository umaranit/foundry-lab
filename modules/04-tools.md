# Module 4: Tools — Built-in (Portal)

**Objective:** Add the Code Interpreter built-in tool to the agent via the Foundry portal so it
can analyze the sample discrepancy trend dataset. (File Search was already added in Module 3.)

**Estimated duration:** 15 minutes

**Prerequisites:** Module 3 complete — the **Invoice Discrepancy Assistant** has File Search
attached and grounds policy questions with citations to `invoice-po-policy.md`.

## Steps

### 1. Recap: File Search is already attached

Module 3 attached **File Search** (Foundry IQ) so the agent can ground policy answers in
`invoice-po-policy.md`. That tool stays exactly as configured — this module doesn't touch it. You
are only *adding* a second built-in tool: **Code Interpreter**, so the agent can run code against
a dataset rather than just retrieving and citing text.

### 2. Add the Code Interpreter tool

1. In your Foundry project, return to **Agents** and open the **Invoice Discrepancy Assistant**.
2. In the agent's configuration, find the **Tools** section and select **+ Add** > **Code
   interpreter**.
3. Confirm Code Interpreter now appears alongside File Search in the agent's tools list.
4. Save the agent configuration.

> **What Code Interpreter does:** unlike File Search (which retrieves and cites relevant text
> chunks), Code Interpreter gives the model a sandboxed environment to write and execute Python
> against files you attach to it — useful for aggregation, filtering, and simple analysis that a
> text-retrieval tool can't do, such as "which category increased over a date range?"

### 3. Upload the discrepancy trend dataset to Code Interpreter

1. Still in the Code Interpreter tool's configuration, select **+ Upload file** (Code
   Interpreter keeps its own attached files, separate from the File Search knowledge source from
   Module 3).
2. Choose [`discrepancy-trends.csv`](/data/discrepancy-trends.csv) from your local clone of
   this lab.
3. Wait for the upload to show a **Ready** (or equivalent "processed") status before continuing.
4. Save the agent configuration again if the portal requires a separate save after the upload.

> **About this dataset:** `discrepancy-trends.csv` contains a year's worth of historical
> discrepancy events (date, discrepancy type, vendor, dollar amount, and resolution status) — see
> [data/README.md](/data/README.md) for details. It's a separate file from the PO records used
> in Module 5; this one is for trend analysis, not looking up an individual PO.

### 4. Update the system instructions

Open the agent's **Instructions** field and add a short line describing the new capability, right
after the existing guideline about citing policy rules:

```text
- You can analyze historical discrepancy trend data (dates, discrepancy types, vendors, amounts,
  resolution outcomes) using the attached code interpreter tool when asked a trend or
  aggregation question — run code against the data rather than estimating an answer from memory.
```

Save the agent.

### 5. Test in the playground

1. Open the agent's **Playground**.
2. Ask a trend question that requires aggregation across the dataset:

   ```text
   Which discrepancy type increased the most over the last few quarters?
   ```

   Confirm the response identifies **quantity mismatches** as the type trending upward (driven by
   a recurring vendor), and that the playground shows a code-execution step (not just a text
   answer) — most portal versions surface this as an expandable "Code" or "Tool call" panel.

3. Try a follow-up analytical question in the same conversation:

   ```text
   Break that down by vendor — who's driving the increase?
   ```

   Confirm the agent re-runs analysis against the same attached file (it shouldn't ask you to
   re-upload it) and names the vendor with the most quantity-mismatch events.

4. Ask one more question to confirm the agent distinguishes the two built-in tools correctly:

   ```text
   What's our policy on quantity mismatch escalation, and has that vendor's over-billing been
   escalated as often as policy requires?
   ```

   This should trigger **both** tools in one response: File Search for the Section 4 policy rule,
   and Code Interpreter for the actual escalation counts in the data.

### 6. Note how this differs from Modules 5 and 6

Code Interpreter and File Search are **built-in tools** — you attached them declaratively in the
portal, with no code of your own. Starting in Module 5, you'll add a **custom function tool**
(PO lookup) that you define and implement yourself, and in Module 6 an **MCP tool** backed by a
server you run locally. Both of those require creating a new version of the agent in code,
whereas this module's Code Interpreter tool is portal-only, like File Search in Module 3.

## ✅ Checkpoint / Validate

- [ ] **Code Interpreter** appears in the agent's Tools list alongside **File Search**.
- [ ] `discrepancy-trends.csv` is uploaded to Code Interpreter and shows a **Ready**/processed
  status.
- [ ] Asking "Which discrepancy type increased the most over the last few quarters?" returns
  **quantity mismatches**, backed by a visible code-execution step rather than a guessed answer.
- [ ] A follow-up analytical question in the same conversation is answered without re-uploading
  the file.

## 🚀 Try it yourself

Ask the agent a chart-style question, for example:

```text
Can you show me a chart of discrepancy counts by quarter, broken out by discrepancy type?
```

Confirm Code Interpreter can generate and return an image/chart artifact in the playground (most
portal versions render this inline or as a downloadable attachment) — this is a good illustration
of why Code Interpreter is a different kind of tool from File Search, which only ever returns
text and citations.

## ⚠️ Troubleshooting

- **Agent answers the trend question from "general knowledge" instead of running code:** The file
  may not actually be attached to Code Interpreter yet (uploads to File Search and Code
  Interpreter are separate) — recheck Step 3 and confirm the file shows under the Code
  Interpreter tool specifically, not just under File Search.
- **No code-execution step visible in the playground:** Some portal versions collapse this into
  an expandable "Code" or "Tool call" panel rather than showing it inline by default — look for a
  toggle near the response before assuming the tool wasn't used.
- **Agent gets the trend direction wrong:** Confirm you uploaded the exact
  `discrepancy-trends.csv` from this lab's `data/` folder and that it finished processing before
  you asked the question — a partially indexed or stale file can produce an incomplete analysis.
- **Upload fails or the file shows an error status:** Confirm the file is a valid CSV and under
  your portal's per-file size limit (this sample file is well under a megabyte, so a failure
  usually indicates a transient upload issue — retry the upload).
