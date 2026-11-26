---
name: orbita-gl-audit
description: Detects accounting posting errors made by Orbita Tech's accountants in Odoo 19 (كشف أخطاء التسجيل المحاسبي) — duplicate bills/invoices, wrong VAT, manual entries hitting control accounts, lines without partner, wrong-direction balances, back-dated or future entries, stale suspense/outstanding balances, old drafts, customers in credit and vendors in debit, missing analytic, unlocked periods — and scores each accountant. Produces an Arabic Excel exception workbook with the exact entries to fix and suggested corrections. Use this whenever the user asks "اكتشف الأخطاء المحاسبية", "مراجعة قيود المحاسبين", "أخطاء التسجيل", "مراجعة دفتر الأستاذ", "قيود غلط", "audit the journal entries", "GL review", "pre-close review", "فواتير مكررة", "ضريبة غلط", or before month-end close or VAT filing, even if they don't name it.
---

# Orbita GL audit — accountant posting errors

Reader: the Financial Manager reviewing his team's work before close. Value comes from precise, fixable exceptions (entry number, user, amount, why it's wrong, what the right entry is), not from a long list of noise. Rank by amount × likelihood of real error, and say plainly when a check returned nothing.

## Workflow
1. **Period**: default = current month to date plus open items from prior periods (drafts, suspense, duplicates look back 12 months). State it.
2. **Data** via AnythingMCP, same rules as `orbita-collections-aging` (one confirmation for read-only pulls; never write; pass `fields`; page; `formatted_read_group` for balances; `odoo_fields_get` on errors). Or use an uploaded Journal Items export.
3. Run the checks in `references/checks.md` (each has domain/logic, threshold, and false-positive guard).
4. Score per user: exceptions count and SAR value by `create_uid` (who entered) — Odoo 19 has no standard "posted by" field; if `write_uid` differs, show both. Normalise by volume (exceptions per 100 entries) before calling anyone the weakest.
5. Workbook (xlsx skill, Arabic, RTL): ملخص · حسب_المحاسب · one sheet per check that found something · قيود_تصحيح_مقترحة · فحوص_بلا_نتائج · ملاحظات.
6. Chat summary.

## Core pulls
- `account.move.line` for the period: `move_id, move_name, date, account_id, partner_id, name, debit, credit, balance, journal_id, tax_ids, tax_line_id, analytic_distribution, parent_state, move_type, create_uid, create_date, reconciled, amount_residual, company_id` (page fully; for 12-month look-backs use grouped queries).
- `account.move` for the period: `name, move_type, state, partner_id, ref, invoice_date, date, amount_untaxed_signed, amount_tax_signed, amount_total_signed, journal_id, create_uid, create_date, reversed_entry_id, payment_state`.
- `account.account`: `code, name, account_type, reconcile`. `res.partner` VAT numbers (`vat`, `country_id`) for partners in scope. `res.company`: `fiscalyear_lock_date, tax_lock_date, sale_lock_date, purchase_lock_date, hard_lock_date` (field names differ by version — adapt).

## Chat summary (Arabic by default)
Lead with the single most serious finding (e.g. a probable duplicate vendor bill already paid, VAT under-declared, a large manual entry on receivables with no partner). Then:
1. **ملخص الفحوص**: table — check, count, SAR, severity (عالي/متوسط/منخفض).
2. **أخطر 5 قيود** with number, user, amount, problem, fix.
3. **حسب المحاسب**: count, value, rate per 100 entries, most frequent error type — framed as training/control need, not blame.
4. **أثر على الضريبة والإقفال**: what must be fixed before the VAT return / month close.
5. **ضوابط مقترحة** (3–5 specific Odoo settings): lock dates, restrict manual entries on control accounts (account "Allow reconciliation"/deprecate posting with journal restrictions), vendor bill reference uniqueness warning, mandatory analytic plans, review of drafts weekly, segregation (see `orbita-odoo-access-review`).

Never post corrections. Suggested correcting entries are proposals for the accountant to post. Use the user's confidence labels when their preferences ask for them; label checks with high false-positive rates as [مرجّح] or [تخمين].
