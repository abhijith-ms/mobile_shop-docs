# Session Handoff — paste this as your first message in the new chat

## Project
Mobile Shop ERP for a second-hand + new phone retailer in Bahrain, built as a custom
Frappe app (`mobile_shop`) on ERPNext 15. Full history/decisions/lessons are in
`PROJECT_PLAN.md` in the project folder — reference it for anything not covered here.

Repo: `~/Documents/Work/mobile_shop/frappe-bench/apps/mobile_shop` (git-tracked, branch `develop`)
Site: `mobileshop.local`

## What's fully done and verified
- Phone / Customer doctypes, ERPNext core Supplier reused (not custom)
- Purchase Entry, Sales Entry workflows
- Bahrain Profit Margin Scheme VAT engine for used phones (verified against NBR guide)
- New/unused phone standard VAT (10%, Inclusive/Exclusive toggle)
- PMS Customer Invoice + Standard VAT Invoice + Self-Billed Purchase Invoice print formats
- Role permissions: Mobile Shop Staff vs Admin/System Manager (purchase_price, margin,
  vat_amount, net_profit hidden from Staff everywhere, including reports)
- Camera-based IMEI barcode/QR scanning (html5-qrcode)
- IMEI validation: hard reject on length (14-16 digits), soft warning on Luhn checksum fail
- Simplified workspace dashboard with 3 live Number Cards + grouped shortcuts
- Reports done (7 of 8): IMEI History, Sales, Purchase, Customer, Supplier, Inventory, Profit
- **VAT Report is the one report still not built** — this is the next task

## Immediate next step
Build **VAT Report** (Admin-only, same hard-permission-guard pattern as Profit Report).
Likely needs both Sales Entry AND Purchase Entry data (sales VAT collected vs purchase-side
context) for a liability picture. Process to follow, no exceptions:
1. Run `frappe.db.get_table_columns("Sales Entry")` and `("Purchase Entry")` in bench console,
   paste REAL output before designing anything.
2. Design columns/SQL/filters, get it reviewed before implementing.
3. Admin-only: JSON roles = only Mobile Shop Admin + System Manager, PLUS a hard
   `has_admin_role()` check as literal first line of `execute()` that throws
   `frappe.PermissionError` if false.
4. Implement, migrate, then user manually verifies BOTH roles in browser (Staff should get
   "You don't have access", Admin should see correct data) — never trust "should work."

## Critical lessons to carry forward (full detail in PROJECT_PLAN.md)
1. **Workspace and Number Card are "standard" doctypes** — they sync from module-level JSON
   files on migrate, but that sync is INSERT-ONLY (never updates an existing record). If
   also listed in `hooks.py`'s `fixtures`, a SECOND competing sync mechanism can silently
   overwrite real data with stale content. Both were removed from fixtures; stale
   `fixtures/workspace.json` and `fixtures/number_card.json` were deleted. Any further
   Workspace/Number Card fix must be applied directly via `frappe.db.set_value()` +
   `commit()` in console — editing the file alone won't fix an existing record.
2. **Report folder/file names are NOT arbitrary** — Frappe derives the expected Python
   module path from the Report's `name` field, scrubbed (lowercase, spaces→underscores).
   "IMEI History Report" → folder `imei_history_report`, not a shorter name. Wrong folder
   name → `ModuleNotFoundError` at runtime even if the JSON name is correct.
3. **Never trust an AI agent's self-report as verification.** Every real bug in this project
   was caught by the user actually clicking through the browser as both Staff and Admin —
   not by the agent saying "tests pass" or "verified successfully."
4. **Script Report columns with `"permlevel": 1` enforce NOTHING on their own** — Script
   Reports run raw SQL directly, bypassing document-level field permissions. Sensitive
   fields must be omitted entirely from both the columns list AND the SQL SELECT via an
   explicit `is_admin` check — never fetched, never a NULL placeholder.
5. **Always verify schema before writing report/query code** — run
   `frappe.db.get_table_columns(doctype)` and show the REAL output first. A rushed batch
   attempt once fabricated fields like `grade`, `posting_date`, `cr_number`,
   `mobile_number` that don't exist anywhere in this schema — caused real time loss.
6. **Bahrain Dinar uses 3 decimal places (fils)**, not Frappe's Currency default of 2 —
   `"precision": "3"` required on all BHD currency fields.
7. **Naming series need dots**: `PE-.YYYY.-.MM.-.#####`, not `PE-YYYY-MM-#####`.
8. **Never `rm -rf` near `apps/*/public/` or other source directories** — an agent did this
   once and deleted working JS files (recovered via `git restore` since the repo was
   git-tracked — commit often).
9. Two git repos exist: inner (`apps/mobile_shop/`, actively used, has real history) and
   outer (`~/Documents/Work/mobile_shop/`, barely used). The inner one is what matters.

## Small unfinished items (safe, no dependency, just never got to them)
- Customer doctype: add optional `address` field.
- Sales Entry: add search by brand/model (currently IMEI-only search).

## Open / pending items (not urgent, don't start without confirming)
- **Accountant clarifications pending** (sent, awaiting reply): whether PMS invoices should
  show VAT amount (this would CONTRADICT the NBR guide's "must not show VAT amount" rule —
  do not implement without explicit reconfirmation with NBR), whether the inclusive/exclusive
  toggle should apply to used-phone sales, "Serial Number" field meaning (likely a shared
  SKU/barcode for accessories, NOT unique like IMEI — confirmed via a screenshot of their
  other retail software), what "Supplier invoice" means vs the existing Self-Billed Purchase
  Invoice, and whether a single sale needs multiple line items (phone + accessory + charge).
- **Multi-category expansion** (earbuds, chargers, other electronics) — decided to likely lean
  toward ERPNext's native Item/Serial No/Stock system rather than extending the custom Phone
  doctype, informed by the accountant's described workflow: bulk stock-in by
  model/quantity (no individual scanning at intake) for both new phones and accessories, with
  IMEI captured only at the point of SALE for phones. Not started.
- **Full accounting integration (Phase A of the client's expanded proposal)** — client wants
  P&L, Balance Sheet, GL, AR/AP, which requires migrating custom Sales Entry/Purchase Entry
  to actually post to ERPNext's native accounting ledger (Sales Invoice/Purchase Invoice/
  Chart of Accounts). This is a genuine architecture decision, not a bolt-on. A phased
  delivery plan document was created and given to the user to share with the client
  (`Mobile_Shop_Phased_Delivery_Plan.docx`). Not started — client wants "everything before
  launch," pushback was given that this needs sequencing.
- **Custom/large-touch-point UI** — client wants big touch targets, possibly tablet-oriented.
  Explored ERPNext's native POS screen as a possible starting point; confirmed it requires
  Phase A (Warehouse/Item/Accounting setup) to even render meaningfully. Paused, to revisit
  once Phase A is underway.
- Original PRD's inventory ageing / dead stock / low-stock reports — not started.

## Tools/workflow notes
- Cline + Kimi2.5 via Bedrock is the main build driver. Antigravity free tier is unreliable
  for sustained work (rate limits), better for quick lookups/second opinions.
- ngrok tunnel used for client demos and for testing camera scanning (requires HTTPS):
  `ngrok http 8000 --host-header="mobileshop.local:8000"`
- `bench start` must be running in its own terminal at all times during dev/testing.
