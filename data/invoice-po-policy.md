# Contoso Finance — Invoice & Purchase Order Discrepancy Policy

| | |
|---|---|
| **Document ID** | CF-AP-POL-014 |
| **Owner** | Accounts Payable Operations, Contoso Finance |
| **Effective date** | 2026-01-05 |
| **Version** | 3.2 |
| **Applies to** | All AP analysts, AP supervisors, and Procurement staff processing vendor invoices against purchase orders (POs) |

> This document is the authoritative source for how Contoso Finance AP analysts triage invoice
> discrepancies. It is fictional content created for the Invoice Discrepancy Assistant workshop
> lab and should not be treated as real company policy.

## 1. Purpose and scope

This policy defines how AP analysts identify, classify, and resolve discrepancies between vendor
invoices and their associated purchase orders. It covers four discrepancy types:

1. **Price mismatch** — the invoiced unit price differs from the PO unit price.
2. **Quantity mismatch** — the invoiced quantity differs from the PO (or receiving-report)
   quantity.
3. **Duplicate invoice** — an invoice that appears to have already been submitted and/or paid.
4. **Missing PO** — an invoice references a PO number that does not exist in the PO system, or no
   PO number is provided at all.

Every invoice must be matched against its PO (a "three-way match" against the PO, the receiving
report, and the invoice) before payment is approved. This policy governs what happens when that
match fails.

## 2. General triage workflow

For every flagged invoice, an AP analyst must:

1. Confirm the discrepancy type (price, quantity, duplicate, or missing PO) using the PO lookup
   tool and the vendor's invoice detail.
2. Apply the tolerance thresholds in Sections 3–6 to determine whether the discrepancy can be
   **auto-approved**, **approved with supervisor sign-off**, or must be **escalated and held**.
3. Record the discrepancy, its classification, and the resolution in the AP exceptions log,
   regardless of outcome.
4. Notify the vendor and/or requisitioner only when the resolution path in the relevant section
   below requires it.

Analysts must never approve a payment that exceeds their authority limit (Section 7) regardless
of how small the underlying discrepancy appears.

## 3. Price mismatch handling

A price mismatch exists when the invoiced unit price differs from the PO unit price for the same
line item.

### 3.1 Tolerance thresholds

| Price variance | Action |
|---|---|
| ≤ 2% **and** ≤ $25 total line-item impact | **Auto-approve.** The analyst may approve the invoice at the invoiced amount without further review. |
| > 2% but ≤ 5%, **or** > $25 but ≤ $250 total line-item impact | **Supervisor sign-off required.** The analyst routes the invoice to the AP supervisor queue with the PO and invoice attached. Supervisor must respond within **2 business days**. |
| > 5% **or** > $250 total line-item impact | **Escalate to Procurement.** The analyst holds payment and opens a price-discrepancy case with the Procurement category manager for that vendor. Procurement must respond within **3 business days**. Do not pay any portion of the disputed line until Procurement confirms the correct price. |

Both the percentage and dollar conditions must be evaluated — if either threshold is exceeded,
use the higher tier. For example, a 1% variance on a $50,000 line item is a $500 impact, which
exceeds the $250 escalation threshold even though the percentage is small, so it must be
escalated to Procurement.

### 3.2 Escalation procedure

1. Attach the invoice, the PO, and a side-by-side price comparison to the case.
2. Notify the vendor's Procurement category manager via the AP exceptions queue (not direct
   email) so the case is tracked.
3. If Procurement confirms the PO price is correct, the invoice is returned to the vendor for
   correction (do not short-pay without vendor agreement).
4. If Procurement confirms the invoiced price is correct (for example, a PO that was never
   updated after a negotiated price change), the analyst may approve at the invoiced price and
   note the PO correction request in the case.

## 4. Quantity mismatch handling

A quantity mismatch exists when the invoiced quantity differs from the quantity on the PO and/or
the receiving report.

### 4.1 Tolerance thresholds

| Quantity variance | Action |
|---|---|
| Invoiced quantity ≤ PO/receiving quantity (under-billing) | **Auto-approve** for the quantity actually received; the analyst short-pays the invoice to match the receiving report and documents the reason in the exceptions log. |
| Over-billing ≤ 3% of ordered quantity **and** ≤ 10 units | **Auto-approve** at the received quantity (pay only for goods received), and flag the vendor's account for monitoring if it recurs more than twice in 90 days. |
| Over-billing > 3% of ordered quantity **or** > 10 units | **Hold and escalate** to the requisitioner (the internal employee who created the PO) to confirm whether a partial shipment is still outstanding or the vendor over-shipped. Requisitioner must respond within **2 business days**. |
| No receiving report exists at all | **Escalate to the Receiving/Warehouse team** before any payment decision — do not assume goods were received. |

The AP analyst never pays for quantities that exceed what the receiving report confirms, even if
the requisitioner later authorizes payment for undelivered goods — that authorization must come
from the AP supervisor plus Procurement jointly, and is logged as a policy exception.

### 4.2 Escalation procedure

1. Pull the receiving report and compare line-by-line against both the PO and the invoice.
2. If the requisitioner confirms a legitimate partial/split shipment, update the case with the
   expected delivery date for the remainder and hold the remaining invoice amount until the
   second shipment is received and reported.
3. If the requisitioner cannot explain the over-shipment within the 2-business-day SLA, escalate
   to the AP supervisor, who decides whether to reject the invoice outright or request a credit
   memo from the vendor.

## 5. Duplicate invoice handling

### 5.1 Detection criteria

Treat an invoice as a **probable duplicate** if it matches an existing invoice already in the AP
system on **all** of the following:

- Same vendor, **and**
- Same PO number (or same vendor + same invoice total if no PO is referenced), **and**
- Invoice total within **$1.00** of an existing invoice, **and**
- Submitted within a **60-calendar-day window** of the existing invoice's submission date.

An exact match on the **vendor's invoice number** against an existing record is always treated as
a **confirmed duplicate**, regardless of the 60-day window, unless the vendor confirms in writing
that the invoice number was mistakenly reused.

### 5.2 Resolution procedure

1. **Confirmed duplicate** (matching vendor invoice number): reject the invoice immediately, do
   not route for approval, and notify the vendor's AP contact with the original invoice's
   reference number and payment status.
2. **Probable duplicate** (matches the criteria above but with a different invoice number): place
   the invoice on hold and contact the vendor within **2 business days** to confirm whether this
   is a resubmission, a billing error, or a legitimately separate charge (for example, a
   recurring service invoice that happens to match in amount).
3. If the vendor confirms it is a duplicate, reject it using the same process as a confirmed
   duplicate.
4. If the vendor confirms it is a legitimately separate invoice, clear the hold, note the
   vendor's confirmation in the exceptions log, and proceed with normal three-way matching.
5. If the original invoice has already been **paid**, any confirmed duplicate must also be
   reported to the AP supervisor as a potential overpayment, so recovery can be pursued
   separately from normal duplicate rejection.

## 6. Missing PO handling

A missing-PO condition exists when an invoice references a PO number that does not exist in the
PO system, or when the invoice has no PO number at all.

### 6.1 Thresholds

| Invoice total | Action |
|---|---|
| ≤ $500 | The analyst may request a **retroactive PO** directly from the requisitioner's manager. If approved, process normally once the retroactive PO is issued. |
| > $500 and ≤ $5,000 | A retroactive PO requires sign-off from both the requisitioner's manager **and** the AP supervisor before it may be issued. |
| > $5,000 | **Retroactive POs are not permitted.** The invoice must be rejected and returned to the vendor with instructions to obtain a valid PO before resubmission, per Procurement policy on pre-approval of spend. |

### 6.2 Procedure

1. Confirm the PO truly does not exist (not simply a typo in the PO number) using the PO lookup
   tool — check for transposed digits or an expired/closed PO before treating it as missing.
2. If a closed or expired PO is found that otherwise matches, route to Procurement to determine
   whether it can be reopened rather than treating this as a true missing-PO case.
3. If no PO can be found, apply the thresholds in Section 6.1.
4. All retroactive PO requests must be logged with the business justification provided by the
   requisitioner, regardless of dollar amount.
5. Invoices rejected for exceeding the retroactive-PO limit must include a note to the vendor
   referencing this policy so they understand why a valid PO is required.

## 7. Approval authority limits

| Role | May approve invoices up to |
|---|---|
| AP Analyst | $2,500 per invoice, within auto-approve/short-pay thresholds above |
| AP Supervisor | $25,000 per invoice |
| Procurement Category Manager | $100,000 per invoice (price/quantity discrepancies only) |
| Controller | No limit; required for any invoice above $100,000 or any policy exception |

An analyst may never approve a discrepancy resolution that exceeds their own authority limit,
even if the discrepancy itself (price or quantity variance) falls within an auto-approve
threshold elsewhere in this policy. Route to the next approval level instead.

## 8. Escalation contact summary

| Discrepancy type | First escalation contact | SLA |
|---|---|---|
| Price mismatch (Tier 2) | AP Supervisor | 2 business days |
| Price mismatch (Tier 3) | Procurement Category Manager | 3 business days |
| Quantity mismatch (over-billing, hold) | Requisitioner | 2 business days |
| Quantity mismatch (no receiving report) | Receiving/Warehouse team | 2 business days |
| Duplicate invoice (probable) | Vendor AP contact | 2 business days |
| Missing PO ($500–$5,000) | Requisitioner's manager + AP Supervisor | 3 business days |
| Missing PO (> $5,000) | Vendor (invoice rejected, no internal escalation) | N/A |

## 9. Documentation and audit requirements

- Every discrepancy, regardless of resolution path, must be logged in the AP exceptions log with
  the discrepancy type, amount, resolution, and approver.
- Auto-approved discrepancies are subject to quarterly sampling audit by Internal Audit; analysts
  must be able to show the tolerance calculation used.
- Policy exceptions (any resolution that deviates from the thresholds in this document) require
  written justification and Controller sign-off, and are reviewed monthly by AP Operations.

## 10. Revision history

| Version | Date | Summary |
|---|---|---|
| 3.2 | 2026-01-05 | Clarified that both percentage and dollar thresholds must be evaluated for price mismatches; added closed/expired PO guidance to missing-PO procedure. |
| 3.1 | 2025-07-14 | Added quantity mismatch monitoring flag for recurring over-billing vendors. |
| 3.0 | 2025-01-10 | Full rewrite introducing the current tolerance tables and approval authority limits. |
