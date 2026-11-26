---
name: orbita-credit-control
description: Credit control for Orbita Tech on Odoo 19 — lists customers over their credit limit (الحد الائتماني) or past their credit period (فترة الائتمان / فواتير متأخرة) who still have open or new sales orders, and explains and configures how to block them in Odoo (credit limit settings, partner blocking warnings, and an Automation Rule that refuses to confirm a sales order for a blocked customer, with a finance override). Use this whenever the user asks "ازاي اوقف العملاء اللي تعدوا الحد الائتماني", "إيقاف البيع للعميل المتأخر", "credit hold", "credit limit", "credit period", "العملاء المتجاوزين", "حظر العميل", "لا تبيع لعميل عليه متأخرات", "منع تأكيد أمر البيع", or wants a list of customers who should be on hold, even if they don't say "credit control".
---

# Orbita credit control (credit limit + credit period)

Reader: the Financial Manager, who must decide who goes on hold and needs Odoo to enforce it so sales cannot bypass it. Two separate tests, because they fail differently:
- **Credit limit**: exposure = receivable balance + confirmed-but-uninvoiced sales orders + the new order. Above limit → hold.
- **Credit period**: any posted invoice past due by more than the grace days → hold, whatever the limit. This is the test that catches good-volume customers who simply don't pay.

## Two modes — pick from the request

**A. "Who should be on hold?" (report).** Read-only. Pull via AnythingMCP (same rules as `orbita-collections-aging`: one confirmation for read-only pulls, never write, pass `fields`, page with limit 500, `formatted_read_group` where possible):
1. `res.partner` (companies, customer rank > 0): `name, user_id, credit_limit, use_partner_credit_limit, property_payment_term_id, credit, credit_to_invoice, sale_warn, sale_warn_msg`. If a field errors, `odoo_fields_get` and adapt (Odoo 19 renamed some warning fields).
2. Company default: `res.company` `account_use_credit_limit, account_default_credit_limit`.
3. Overdue: `formatted_read_group` on `account.move`, domain `move_type=out_invoice, state=posted, payment_state in [not_paid, partial], invoice_date_due < AS_OF − GRACE`, group by `commercial_partner_id`, aggregates `amount_residual_signed:sum, invoice_date_due:min, __count`.
4. Open sales: `sale.order` with `state in [draft, sent, sale]` per partner: `name, state, amount_total, amount_to_invoice, user_id, date_order` (drafts/quotations = orders about to be confirmed).

Default `GRACE` = 0 days unless the user gives one; state it. Build the workbook (xlsx skill, Arabic, RTL): ملخص · متجاوز_الحد · متأخر_السداد · كلاهما · أوامر_مفتوحة_لعملاء_موقوفين · بدون_حد_ائتماني (customers with no limit set — a control gap in itself) · ملاحظات.
For each customer: limit, receivable, uninvoiced SO, exposure, % of limit, oldest overdue days, overdue amount, salesperson, open quotations, recommended status (إيقاف / تحذير / سليم).

Summary: lead with the open quotations/orders that belong to customers who should already be on hold (that's money about to go out the door). Then counts and amounts per category, top 5 by exposure, salesperson with most at-risk customers, and customers with no limit.

**B. "How do I stop them?" (configuration).** Explain, give exact steps, and supply the automation code from `references/odoo-blocking-setup.md`. Do not make the change in Odoo yourself unless the user explicitly orders it and confirms the exact write; configuration of server actions is normally done by the admin in the UI.

## Key facts to state (confidence)
- [مؤكد] Standard Odoo credit limit (Accounting settings › Customer Invoices › Sales Credit Limit) only **warns** on the quotation/invoice; it does **not** block confirmation. That is why sales can still confirm.
- [مؤكد] Standard Odoo has **no** credit-period (overdue) block at all.
- [مرجّح] The partner "Sales Warning" with a *Blocking Message* blocks selecting the customer on a new quotation, but it is manual per customer and someone must remember to remove it; in Odoo 19 check the option still exists on the partner form before recommending it.
- [مرجّح] Reliable enforcement without custom modules = an **Automation Rule** on `sale.order` that raises an error on confirmation when either test fails, with an override for Accounting Administrators. Optionally the same check on delivery validation (`stock.picking`) so goods don't leave even if an order slipped through.

Recommend the policy alongside the technique: limits set for every credit customer (from payment history, not sales enthusiasm), grace days agreed with management in writing, who can override and how it is logged, and a weekly hold list sent to sales. Flag the risk: a hard block without an override process will be bypassed by creating a new partner — so also restrict partner creation to finance or require VAT number uniqueness.
