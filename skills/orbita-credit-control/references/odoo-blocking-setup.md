# Blocking over-limit and overdue customers in Odoo 19

## Contents
1. Step 1 – turn on credit limits (standard)
2. Step 2 – Automation Rule that blocks confirmation (credit limit + credit period)
3. Step 3 – optional: block delivery validation
4. Step 4 – manual hold for a single customer
5. Testing checklist and known traps

Tell the user which steps need Odoo admin rights (Settings) and that Automation Rules require the `base_automation` module (Settings › Technical needs developer mode: Settings › Activate the developer mode).

## 1. Turn on credit limits

1. Accounting › Configuration › Settings › Customer Invoices › **Sales Credit Limit** → enable, set a default limit (0 = no default).
2. Each customer: Contacts › customer › **Accounting** tab (Invoicing tab in some editions) › tick **Partner Limit** and enter the amount.
3. Payment terms: customer › Sales & Purchase tab › Payment Terms (e.g. 30 days). The credit-period test uses invoice due dates, which come from this field, so customers without terms default to "due immediately" — fix those first or they will all appear overdue.

Effect: Odoo shows a yellow warning on the quotation and invoice. It does not stop confirmation. Steps 2–3 add the block.

## 2. Automation Rule: block confirmation

Settings › Technical › Automation › **Automation Rules** › New:
- **Model**: Sales Order (`sale.order`)
- **Trigger**: *State is set to* → **Sales Order** (i.e. state = `sale`). (If your build only offers "On save", use Before Update Domain `state in (draft, sent)` and Apply On `state = sale`.)
- **Action**: Execute Code, paste:

```python
# Orbita – credit hold on sales order confirmation
# Blocks if: (1) any posted customer invoice is overdue beyond GRACE_DAYS, or
#            (2) receivable + other uninvoiced confirmed orders + this order > credit limit.
# Accounting Administrators can confirm anyway (override).
GRACE_DAYS = 0          # agreed grace period after due date
MIN_OVERDUE = 1.0       # ignore rounding residues below this amount (SAR)

if not env.user.has_group('account.group_account_manager'):
    today = datetime.date.today()
    cutoff = today - datetime.timedelta(days=GRACE_DAYS)
    for order in records:
        partner = order.partner_id.commercial_partner_id

        overdue = env['account.move'].sudo().search([
            ('commercial_partner_id', '=', partner.id),
            ('move_type', '=', 'out_invoice'),
            ('state', '=', 'posted'),
            ('payment_state', 'in', ['not_paid', 'partial']),
            ('invoice_date_due', '<', cutoff),
            ('company_id', '=', order.company_id.id),
        ])
        overdue_amt = sum(overdue.mapped('amount_residual_signed'))
        if overdue_amt >= MIN_OVERDUE:
            oldest = min(overdue.mapped('invoice_date_due'))
            raise UserError(
                "لا يمكن تأكيد أمر البيع %s: العميل %s لديه فواتير متأخرة بمبلغ %.2f منذ %s (%d فاتورة).\n"
                "Cannot confirm %s: customer has overdue invoices of %.2f since %s. "
                "Contact Finance for approval."
                % (order.name, partner.name, overdue_amt, oldest, len(overdue),
                   order.name, overdue_amt, oldest))

        if partner.use_partner_credit_limit:
            limit = partner.credit_limit
        else:
            limit = order.company_id.account_default_credit_limit if order.company_id.account_use_credit_limit else 0.0
        if limit > 0:
            other_orders = env['sale.order'].sudo().search([
                ('partner_id', 'child_of', partner.id),
                ('state', '=', 'sale'),
                ('id', '!=', order.id),
                ('company_id', '=', order.company_id.id),
            ])
            uninvoiced = sum(other_orders.mapped('amount_to_invoice'))
            exposure = partner.credit + uninvoiced + order.amount_total
            if exposure > limit:
                raise UserError(
                    "لا يمكن تأكيد أمر البيع %s: إجمالي مديونية العميل %s سيصبح %.2f ويتجاوز الحد الائتماني %.2f.\n"
                    "Cannot confirm %s: exposure %.2f exceeds credit limit %.2f. Contact Finance."
                    % (order.name, partner.name, exposure, limit, order.name, exposure, limit))
```

Notes for the user:
- [مرجّح] The error rolls back the confirmation; the order stays a quotation.
- `partner.credit` is the receivable balance in company currency; `order.amount_total` is in the order currency — for foreign-currency orders convert with `order.currency_id._convert(order.amount_total, order.company_id.currency_id, order.company_id, today)`.
- Override group: replace `account.group_account_manager` with a dedicated group if the CFO wants sales managers to approve some cases; never give the override to salespeople.
- Odoo logs who confirmed in the chatter, so overrides are auditable. A stronger option is a "Credit approved until" date field on the partner (Studio) checked in the code.

## 3. Optional: block delivery validation

Second rule on **Transfer** (`stock.picking`), trigger *State is set to* Done, apply on `picking_type_code = outgoing`, code: run the same overdue test on `record.partner_id.commercial_partner_id` for each record in `records` (skip returns: `picking_type_code == 'outgoing'` and not `record.return_id`). This stops goods leaving for an order that was confirmed before the customer went overdue.

## 4. Manual hold for one customer

Customer › Internal Notes / Sales & Purchase tab › **Sales Warning** → *Blocking Message* with the reason. [مرجّح] Available in Odoo 17/18; confirm on the Odoo 19 form. Remember to remove it after payment — that's why the automation in step 2 is preferred.

## 5. Testing and traps

Test in a copy/staging DB or with a test customer: (a) customer with one overdue invoice → confirm should fail; (b) same as Accounting Administrator → succeeds; (c) customer under limit, no overdue → succeeds; (d) customer whose new order pushes exposure over limit → fails.

Traps:
- Unapplied customer payments/credit notes make a customer look overdue while net balance is zero. Clean them (reconcile) before go-live, or test net `partner.credit` ≤ 0 → allow.
- Customers created twice (duplicates) bypass the block. Restrict contact creation and check duplicates by VAT number.
- Down-payment invoices that are due but intentionally unpaid until delivery will block. Decide policy.
- Multi-company: the code filters on the order's company.
