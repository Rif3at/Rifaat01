# Roles, conflicts and hygiene checks (Odoo 19)

## Key roles (xml ids — verify via ir.model.data; names in Arabic UI may differ)
| Role | xml id |
|---|---|
| Settings / System admin | `base.group_system` (also `base.group_erp_manager` = Access Rights) |
| Accounting Administrator | `account.group_account_manager` |
| Accountant / full accounting features | `account.group_account_user` |
| Invoicing (billing) | `account.group_account_invoice` |
| Auditor / read-only | `account.group_account_readonly` |
| Validate bank accounts | `account.group_validate_bank_account` |
| Sales user / all docs / admin | `sales_team.group_sale_salesman`, `sales_team.group_sale_salesman_all_leads`, `sales_team.group_sale_manager` |
| Purchase user / admin | `purchase.group_purchase_user`, `purchase.group_purchase_manager` |
| Inventory user / admin | `stock.group_stock_user`, `stock.group_stock_manager` |
| HR / Payroll officer & admin | `hr.group_hr_user`, `hr.group_hr_manager`, `hr_payroll.group_hr_payroll_user/manager` (if installed) |
| Contact creation | `base.group_partner_manager` |

Groups imply others (admin ⇒ user). Expand implied groups before testing conflicts (`implied_ids` on res.groups, recursively).

## Segregation-of-duties conflicts
| # | Combination | Risk | Severity |
|---|---|---|---|
| 1 | Create/edit vendor or its bank account (`group_partner_manager` or `group_validate_bank_account`) + post vendor bills + register payments (`group_account_invoice`+) | Fictitious vendor or IBAN swap → payment to self | H |
| 2 | Purchase manager (approve PO) + Inventory user (validate receipt) + Billing | Order, "receive" and bill without independent check | H |
| 3 | Accounting Administrator for anyone other than the Financial Manager | Can change lock dates, taxes, journals, delete/reset entries | H |
| 4 | Settings admin for finance or sales staff | Can grant themselves any right; changes untraceable to the business | H |
| 5 | Sales manager + Invoicing (credit notes) | Issue credit notes to own customers to hide collections or inflate commission | M |
| 6 | Inventory admin + Accounting | Stock adjustments and the valuation entries by the same person | M |
| 7 | Payroll manager + payments | Change salary and pay it | H |
| 8 | Salesperson with "All documents" + delete rights on quotations | Low; visibility only | L |
| 9 | Cash journal user who also reconciles bank | Teeming and lading | M |

For a small team, a conflict may be unavoidable: then require a compensating control (listed in SKILL.md) and record it as accepted risk signed by management.

## Hygiene checks
- Active users with `login_date` > 60 days ago or never logged in.
- Active users linked to an archived employee or with `departure_date` set.
- Generic/shared logins.
- More than 2 users with `base.group_system`.
- Finance users (any account group) without 2FA, if `totp_enabled` is available.
- Internal users created only to receive email (should be contacts or portal).
- API keys on non-technical users.
- Users with access to more companies than they work for.
- Licence angle: inactive internal users still cost a paid Odoo seat — quantify count × seat price if the user gives the price.
