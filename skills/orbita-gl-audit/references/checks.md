# GL audit checks (Odoo 19)

Severity: H = likely money/tax impact, M = misstatement or control gap, L = hygiene. Each check lists the guard against false positives — apply it before reporting.

## A. Duplicates
1. **Duplicate vendor bills (H)** — `in_invoice` posted, same `partner_id` and same normalised `ref` (strip spaces, case, leading zeros); or same partner + same `amount_total_signed` within 10 days. Guard: exclude pairs where one is reversed (`reversed_entry_id`) or a credit note offsets. Flag if both are paid (`payment_state` paid/in_payment) — that's cash to recover.
2. **Duplicate customer invoices (M)** — same partner, amount, and invoice_date ± 3 days. Guard: recurring subscriptions with equal amounts in different months are fine.
3. **Duplicate payments (H)** — `account.payment` same partner, amount, date ± 3 days, both posted.

## B. VAT (KSA, 15%)
4. **Taxable lines with no tax (H)** — income/expense lines on invoices/bills with empty `tax_ids` where partner country = SA. Guard: exempt/zero-rated services and non-VAT expenses (salaries, government fees, bank charges, GOSI) — map by account and exclude.
5. **Tax ≠ 15% of base (H)** — per move: |amount_tax − 0.15 × taxable base| > 1 SAR. Guard: mixed-rate invoices.
6. **Input VAT claimed without vendor VAT number (H)** — bills with tax lines where vendor `vat` is empty or not 15 digits starting and ending with 3. Under ZATCA input VAT needs a valid tax invoice.
7. **VAT period mismatch (M)** — `invoice_date` and accounting `date` in different VAT quarters.
8. **Manual entries on VAT accounts (M)** — `move_type = entry` hitting tax accounts, excluding the closing/settlement entry.

## C. Control accounts and partners
9. **Receivable/payable lines without partner (H)** — lines on `asset_receivable` / `liability_payable` with empty `partner_id`. Breaks aging and statements.
10. **Manual entries on receivable/payable/bank (M)** — `move_type = entry`, journal type general, on those accounts. Guard: opening balances, approved write-offs, reclassifications with narration — list but rate L if narration explains.
11. **Wrong-direction balances (M)** — customers with net credit balance and no open invoices (advances misclassified); vendors with net debit balance; expense accounts with net credit, revenue accounts with net debit for the period. Guard: refunds and reversals.
12. **Expense booked to balance sheet or vice versa (M)** — vendor bill lines on asset accounts below a capitalisation threshold (default 5,000 SAR), or on fixed-asset accounts without an asset record; large amounts on miscellaneous expense.

## D. Dates and status
13. **Back-dated entries (M)** — `date` more than 30 days before `create_date`, or into a period already reported (before the last VAT quarter end).
14. **Future-dated entries (M)** — `date` > today.
15. **Old drafts (L/M)** — `state = draft` older than 15 days, by user and amount; drafts are invisible in reports.
16. **Frequent reversals/cancellations by user (L)** — users with reversal rate > 5% of their entries: a training signal.
17. **Lock dates (M)** — fiscal/tax lock date not set or older than the last closed month/quarter.

## E. Clearing and suspense
18. **Stale suspense / outstanding receipts & payments / bank suspense (H)** — unreconciled lines older than 30 days on these accounts, aged; balance not explained = unrecorded cash or posting error.
19. **Unreconciled bank lines (M)** — `account.bank.statement.line` not reconciled older than 15 days (cross-reference the `orbita-bank-reconciliation` skill).
20. **Employee custody/advances (M)** — AR-Staff (103000) balances older than 60 days by employee.

## F. Patterns
21. **Round-amount manual entries ≥ 10,000 SAR (L)** — `entry` moves with amounts multiple of 1,000 and no narration.
22. **Weekend/after-hours posting (L)** — create_date on Friday/Saturday or 22:00–05:00 (Riyadh time, UTC+3). Only report together with another flag.
23. **Missing analytic (L/M)** — income/expense lines without `analytic_distribution` if the company uses departments/projects (check whether ≥ 50% of lines have one; if not, skip and say analytics aren't used).
24. **Negative stock (M)** — `stock.quant` quantity < 0: valuation and COGS wrong.
25. **Cash account negative (H)** — running balance of cash journals below zero on any day.

## Suggested correction format
For each H/M item propose: entry to reverse or amend, the correcting lines (Dr/Cr, account code, amount, partner), and who should post it. Never post.
