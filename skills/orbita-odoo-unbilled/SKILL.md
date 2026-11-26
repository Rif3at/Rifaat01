---
name: orbita-odoo-unbilled
description: Finds Orbita Tech's unbilled operations in Odoo 19 — sales orders delivered (أوامر بيع تم صرفها/تسليمها) but not invoiced, purchase orders received but with no vendor bill (GRNI), and the reverse leaks (invoiced but not delivered, billed but not received) — reconciles them to the stock interim accounts, and produces an Arabic Excel workbook with aging, owner and action per line. Use this whenever the user asks in any wording about "أوامر بيع تم صرفها ولم تفوتر", "مسلّم غير مفوتر", "فواتير لم تصدر", "مشتريات مستلمة ولم تسجل فاتورتها", "استلام بدون فاتورة مورد", "GRNI", "unbilled revenue", "delivered not invoiced", "received not billed", "To Invoice", "Waiting Bills", month-end cut-off or accruals for goods/services, even if they do not name the report.
---

# Orbita unbilled sales & unbilled purchases (Delivered-not-invoiced / GRNI)

The reader is Orbita Tech's Financial Manager (Riyadh, SAR, Odoo 19). Two risks drive this report, and the summary should be framed around them:
- **Sales delivered and not invoiced** = revenue and VAT leakage, cash delayed, and under ZATCA the tax point can arise at delivery, so an uninvoiced delivery can also be a late VAT invoice.
- **Purchases received and not billed** = an unrecorded liability (GRNI). Month-end P&L and payables are understated until the bill is booked or accrued.

## Workflow

1. **Dates.** `as_of` = today unless told otherwise. State it in the first line. Don't ask.
2. **Get data.** Source order: an uploaded export if one is given; otherwise Odoo via the AnythingMCP connector, using the same rules as the `orbita-collections-aging` skill (all Odoo tools run through `anythingmcp_run_write_tool` and need one confirmation — ask once for the read-only pulls listed below; never write to Odoo from this skill; always pass `fields`; count first, page with limit 500; on `Invalid field` run `odoo_fields_get` and adapt). Add `["company_id","=",ID]` if multi-company.
3. **Compute** the four registers below at line level, in company currency.
4. **Tie out** to the GL (section "Control totals"). Lead with the gap if it does not tie.
5. **Build the workbook** with the xlsx skill (Arabic sheet names, RTL, totals as formulas), then write the chat summary.

## Pulls (Odoo 19)

**S1 – sales lines with something to invoice** (`sale.order.line`)
```json
{"domain": [["state","=","sale"],["display_type","=",false],["invoice_status","in",["to invoice","no"]]],
 "fields": ["order_id","order_partner_id","salesman_id","product_id","name","product_uom_qty","qty_delivered","qty_invoiced","qty_to_invoice","price_unit","discount","price_subtotal","untaxed_amount_to_invoice","currency_id","qty_delivered_method"]}
```
Also pull `sale.order.line` with `qty_invoiced > 0` for the reverse check (filter in code: `qty_invoiced > qty_delivered`).

**S2 – delivery dates** (`stock.move`, done, linked to the lines from S1): fields `sale_line_id, date, picking_id, quantity, location_dest_id`. The latest done date per line is the "delivered on" date used for aging. Exclude returns (destination = internal location) from delivered qty only if Odoo has not already netted them (it normally has in `qty_delivered`).

**S3 – products**: `product.product` for every product in S1/P1: `default_code, type, invoice_policy, purchase_method`. In Odoo 19 `type` is `consu` / `service` / `combo` and `is_storable` marks stocked goods.

**P1 – purchase lines** (`purchase.order.line`)
```json
{"domain": [["order_id.state","in",["purchase","done"]],["display_type","=",false]],
 "fields": ["order_id","partner_id","product_id","name","product_qty","qty_received","qty_invoiced","qty_to_invoice","price_unit","discount","price_subtotal","currency_id","date_planned","qty_received_method"]}
```
Restrict to `qty_received != qty_invoiced` in code (fields can't be compared in a domain). Pull receipt dates from `stock.move` with `purchase_line_id` exactly like S2. Get the PO's `user_id` (buyer) from `purchase.order`.

**G1 – control accounts**: `account.account` whose name/code matches Stock Interim (Received), Stock Interim (Delivered), Stock Valuation (Arabic names may be "مخزون مؤقت (مستلم/مسلم)"). Then `formatted_read_group` on `account.move.line` (posted, `account_id in [...]`) grouped by `account_id`, aggregate `balance:sum`. If none exist, the company uses periodic valuation: say so and skip the tie-out.

## The four registers

For each line, value = qty × `price_unit` × (1 − discount/100), converted to SAR with the order's rate if currency ≠ SAR (use `price_subtotal`/`product_uom_qty` ratio when possible to avoid rate guessing). Excl. VAT; show VAT at 15% in a separate column for sales.

| Register | Rule | Meaning |
|---|---|---|
| **مسلّم غير مفوتر** (sales) | `qty_delivered − qty_invoiced > 0` | Main target. Age from last delivery date. |
| **مفوتر غير مسلّم** (sales) | `qty_invoiced − qty_delivered > 0` and product is not `invoice_policy = order` service/prepayment by design | Revenue billed before delivery: deferred revenue / cut-off risk. Separate "by design" (down payments, policy = order) from true exceptions. |
| **مستلم غير مفوتر – GRNI** (purchases) | `qty_received − qty_invoiced > 0` | Unrecorded liability; accrual candidate. Age from last receipt date. |
| **مفوتر غير مستلم** (purchases) | `qty_invoiced − qty_received > 0` | Paying/owing for goods not received: chase supplier or check a missed receipt validation. |

Sales lines where `invoice_policy = order` and nothing is delivered are "ordered, not delivered": list them in a separate info sheet, they are not leakage.

**Services (Orbita is an IT company).** For service lines `qty_delivered` is manual, timesheet or milestone driven (`qty_delivered_method`). A service line with qty_delivered = 0 may simply mean nobody updated delivery. Flag service orders confirmed > 30 days ago with zero delivered and zero invoiced as "تحقق من حالة التنفيذ" rather than counting them as leakage.

Age bands: 0–7, 8–30, 31–60, 61–90, > 90 days since delivery/receipt.

## Control totals

- GRNI register total should ≈ Stock Interim (Received) credit balance. Delivered-not-invoiced at cost should ≈ Stock Interim (Delivered) debit balance (anglo-saxon). Differences point to bills booked to the wrong account, manual entries on interim accounts, or receipts on price differences — list the top movers on that account (manual `entry` moves) as the likely cause.
- If the gap is material (> 1% or > 5,000 SAR) open the summary with it.

## Workbook sheets

الملخص (KPIs, as_of, tie-out) · مسلم_غير_مفوتر · مفوتر_غير_مسلم · مستلم_غير_مفوتر · مفوتر_غير_مستلم · حسب_المندوب (sales registers by salesperson and customer) · حسب_المشتري_والمورد · خدمات_للتحقق · ملاحظات. Each register has: order, customer/vendor, product, qty ordered/delivered(received)/invoiced, gap qty, value SAR, VAT, last movement date, days, age band, owner (salesperson/buyer), suggested action.

## Chat summary structure (Arabic by default)

Lead with the most uncomfortable fact (e.g. "X SAR delivered more than 30 days ago is still not invoiced", or "GRNI does not tie to the interim account by Y"). Then:
1. **الأرقام**: total and count for each of the four registers; share over 30 days.
2. **أكبر 5 أوامر بيع مسلمة غير مفوترة** with customer, salesperson, days.
3. **أكبر 5 استلامات بلا فاتورة مورد** with vendor, buyer, days.
4. **المسؤولية**: which salesperson/buyer holds the most, by amount and by days.
5. **الأثر المحاسبي**: suggested month-end accrual for GRNI (Dr expense/inventory, Cr accrued liabilities — or rely on the interim account if perpetual), and the VAT exposure on uninvoiced deliveries.
6. **إجراءات** (3–5, named): invoice today list; ask vendors for missing bills; close or cancel residual quantities on orders that will never be delivered/billed (Odoo: lock the order or reduce qty) so they stop polluting the list; set a rule — invoice within N days of delivery; for services, update delivered quantities monthly.

Use the user's confidence labels ([مؤكد]/[مرجّح]/[تخمين]) when the user preferences ask for them. Mark currency conversion and the service-delivery interpretation as assumptions.

## Fallback: Excel export

From Odoo UI: Sales › To Invoice › Orders to Invoice (or Sales › Orders, filter "To Invoice", group by Order, export the order lines with the fields above); Purchase › Orders, filter "Waiting Bills", export lines with Received Qty / Billed Qty. Map columns, show the mapping in one line, and note that without stock move dates aging uses order date instead.
