---
name: orbita-odoo-access-review
description: Reviews Odoo 19 user permissions at Orbita Tech (مراجعة صلاحيات المستخدمين) — who has which access rights, segregation-of-duties conflicts (create + approve + pay, purchase + receive, edit vendor bank + pay, sales + credit notes), excess admin/settings rights, inactive or departed users still active, shared logins, users without 2FA, and portal users — and produces an Arabic Excel access matrix with a prioritised list of exact group changes. Use this whenever the user asks about "صلاحيات المستخدمين", "مين عنده صلاحية", "فصل المهام", "segregation of duties", "user access review", "permissions audit", "المحاسب يقدر يعمل دفع", "مستخدمين نشطين", "أمن أودو", or when a fraud/error concern relates to who could post or approve something, even if they don't name an access review.
---

# Orbita Odoo access review

Reader: the Financial & Administrative Manager, accountable for internal control. The output must say exactly which user loses or gains which group, and why — "tighten permissions" is useless.

## Workflow
1. **Data** via AnythingMCP (one confirmation for read-only pulls; never write — even to remove a right; results follow the API key's rights, so if the key isn't an admin, say the review is partial):
   - `res.users` (internal: `share = false`, include inactive): `name, login, active, login_date, company_ids, group_ids` (older name `groups_id` — adapt via `odoo_fields_get`), `totp_enabled` if present, `employee_ids`.
   - `res.groups`: `name, full_name` (or `privilege_id`/`category_id` in Odoo 19), plus `ir.model.data` with `model = res.groups` to map ids → xml ids (e.g. `account.group_account_manager`). Use xml ids for the SoD logic, names for display.
   - `hr.employee` (if installed): `name, user_id, active, departure_date` to find departed staff with active users.
   - Optional: `res.users.apikeys` count per user; `res.users.log` for last activity.
2. Build the matrix: one row per user, one column per key role (see `references/sod-matrix.md`).
3. Apply the conflict rules and hygiene checks in the reference.
4. Workbook (xlsx skill, Arabic, RTL): ملخص · مصفوفة_الصلاحيات · تعارضات_فصل_المهام · مدراء_النظام · مستخدمين_غير_نشطين · تغييرات_مقترحة (user, remove group, add group, reason, risk, priority) · ملاحظات.
5. Chat summary.

## Chat summary (Arabic by default)
Lead with the worst exposure in one sentence (e.g. "User X can create a vendor, change its IBAN, post the bill and register the payment alone"). Then:
1. **الأرقام**: internal users, active, admins (Settings), users with full accounting, conflicts H/M.
2. **تعارضات عالية الخطورة**: user, the roles combined, what fraud/error it enables.
3. **حسابات يجب تعطيلها**: no login > 60 days, departed employees, shared/generic logins (`admin`, `accounts@`, `info@`).
4. **التغييرات المقترحة** in order, ready for the admin to apply in Settings › Users.
5. **ضوابط تعويضية** where a small team makes full segregation impossible: payment approval by a second person in the bank portal, weekly review of vendor bank changes (mail tracking on `res.partner.bank`), lock dates, the `orbita-gl-audit` review monthly, 2FA for all finance users.

Do not apply changes in Odoo from this skill. If the user orders it, list each exact write and get explicit confirmation first. Use the user's confidence labels when their preferences ask for them; group-name mapping from Odoo 19 that couldn't be verified is [مرجّح].
