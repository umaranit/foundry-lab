# Sample Data

> **Status:** All items below have been generated and are ready to use. Items 1, 2, and 4
> (`invoice-po-policy.md`, `po-records.csv`, and `discrepancy-trends.csv`) are used starting in
> Modules 3, 5, and 4, respectively. Item 3's `discrepancy-eval-seed.jsonl` is hand-authored and
> ready for Module 8; its sibling `discrepancy-eval-dataset.jsonl` is generated at run time by
> Module 8's capture script and is not committed here (it depends on live agent responses).

## 1. Invoice & PO policy document ✅ generated

- **Filename:** [`invoice-po-policy.md`](/lab/data/invoice-po-policy.md) (plain Markdown)
- **Used in:** Module 3 (knowledge base / grounding), referenced again in Module 6 (Section 8
  escalation-contact table grounds the MCP tool) and Module 8 (evaluation).
- **Contains:** Contoso Finance's fictional accounts-payable policy for handling invoice
  discrepancies, including:
  - Definitions of the four discrepancy types the agent handles: price mismatch, quantity
    mismatch, duplicate invoice, missing PO.
  - Tolerance thresholds for auto-approval vs. required escalation (e.g., price variance under
    X% can be auto-approved).
  - Step-by-step resolution/escalation procedures for each discrepancy type, including who to
    notify and SLA expectations.
  - Duplicate-invoice detection rules (e.g., matching criteria, time window).
  - Missing-PO handling rules (e.g., when to request a retroactive PO vs. reject the invoice).

## 2. Sample PO records (CSV) ✅ generated

- **Filename:** [`po-records.csv`](/lab/data/po-records.csv) (CSV, 18 rows)
- **Used in:** Module 5 (custom "look up PO by number" function tool, and the client that calls
  it), Module 7 (specialist agents).
- **Contains:** A realistic, fictional set of purchase order records with columns `po_number`,
  `vendor_name`, `item_description`, `unit_price`, `quantity`, `order_date`, and `status` (`Open`,
  `Partially Received`, `Closed - Fully Invoiced`, or `Cancelled`), covering all eight vendors
  used in `discrepancy-trends.csv`. `PO-10234` (used as the example lookup in Modules 2 and 5) is
  included. A handful of records use `Closed - Fully Invoiced` status, which is useful when
  discussing the duplicate-invoice overpayment case in policy Section 5.2. PO numbers are not
  sequential (there are intentional gaps, e.g. no `PO-10243`), so asking about a nearby but
  nonexistent PO number is a realistic way to exercise the missing-PO discrepancy type and the
  transposed-digit guidance in policy Section 6.2 without any extra setup.

## 3. Invoice discrepancy evaluation datasets

- **Filenames:**
  - [`discrepancy-eval-seed.jsonl`](/lab/data/discrepancy-eval-seed.jsonl) ✅ generated — 15
    `{"query": ..., "ground_truth": ...}` objects, one per line. This is the hand-authored input
    file.
  - `discrepancy-eval-dataset.jsonl` — one `{"query": ..., "response": ..., "ground_truth": ...}`
    object per line. This file is *generated*, not committed: Module 8's capture script reads
    `discrepancy-eval-seed.jsonl`, sends each `query` to the live Module 6 hosted agent (the same
    `conversations`/`responses` client pattern as Modules 5–6), records the agent's actual
    `response.output_text`, and writes the enriched triples here. Only this file is a valid input
    to the Evals API — the seed file alone is missing the `response` field every built-in
    evaluator needs.
- **Used in:** Module 8 (evaluation). `discrepancy-eval-seed.jsonl` is the authored input to the
  capture step; `discrepancy-eval-dataset.jsonl` is uploaded via
  `project_client.datasets.upload_file(...)` and scored by the built-in **intent_resolution**,
  **relevance**, and **groundedness** evaluators (verified against
  `mslearn-genaiops/src/evaluators/evaluate_agent.py` — there is no custom evaluator in this
  module).
- **`discrepancy-eval-seed.jsonl` contains:** 15 sample end-user queries about invoice
  discrepancies, each paired with an expected/reference answer (`ground_truth`) covering the key
  facts the response should contain — at least two rows per discrepancy type (price mismatch,
  quantity mismatch, duplicate invoice, missing PO) across their different tolerance tiers, plus
  one row exercising the Module 6 `get_escalation_contact` tool and two "trick" rows (an
  exact-match invoice with no discrepancy at all, and a Section 7 approval-authority-limit case)
  to produce a meaningful aggregate evaluation score. Note that `groundedness`'s `data_mapping`
  reuses this same `ground_truth` value as its `context` parameter — there's no separate
  retrieved-context field in this dataset.

## 4. Discrepancy trend CSV (code interpreter) ✅ generated

- **Filename:** [`discrepancy-trends.csv`](/lab/data/discrepancy-trends.csv) (CSV, 81 rows)
- **Used in:** Module 4 (Code Interpreter tool) only. This is a separate file from
  `po-records.csv` (item 2) — dedicated to historical trend data rather than individual PO
  lookups. Module 7's code-first agents reuse `po-records.csv` for PO lookups, but do not analyze
  this trend file — doing so would require giving one of those agents a code-interpreter-style
  tool, which is out of scope for that module.
- **Contains:** A time-series dataset of historical discrepancy events from January through
  November 2026, with columns `date`, `discrepancy_type` (Price Mismatch / Quantity Mismatch /
  Duplicate Invoice / Missing PO), `vendor_name`, `amount`, and `resolution_status` (e.g.
  Auto-Approved, Supervisor Approved, Escalated - Procurement, Escalated - Requisitioner,
  Escalated - Receiving, Rejected, Pending Vendor Response, Cleared - Not Duplicate, Retroactive
  PO Issued). Quantity-mismatch events trend upward quarter over quarter (4 → 7 → 11), driven
  largely by one recurring vendor, so "which discrepancy type increased last quarter?" and
  "which vendor is driving that increase?" both have clear, data-grounded answers.

