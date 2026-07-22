# Mobile Shop ERP — Project Conventions

Custom Frappe app (`mobile_shop`) on ERPNext 15, for a second-hand + new mobile phone
retailer in Bahrain. Site: `mobileshop.local`. Repo root: this directory
(`apps/mobile_shop`, git-tracked, branch `develop`).

Full build history and detailed debugging narratives live in `PROJECT_PLAN.md` in this
repo — read it on demand when working on something with prior history (e.g. "check
PROJECT_PLAN.md for context on the Sales Report SQL bug"), not automatically every
session.

## Current state (update this section as work progresses)

**Done and verified**: Phone/Customer doctypes, ERPNext core Supplier (reused, not
custom), Purchase Entry, Sales Entry, Bahrain Profit Margin Scheme VAT engine (verified
against the actual NBR guide), New/Used phone VAT branching (standard 10%
Inclusive/Exclusive vs PMS margin scheme), 3 print formats, role permissions
(Mobile Shop Staff vs Admin/System Manager), camera IMEI scanner, IMEI validation
(hard reject on length, soft warn on Luhn), workspace dashboard with 3 Number Cards,
8 of 8 planned reports (IMEI History, Sales, Purchase, Customer, Supplier, Inventory,
Profit, VAT — committed 0e8efcc; Staff-blocked/Admin-full-access confirmed manually
in browser), Sales Entry search by brand/model (committed 57e6ea7 — `search_phones()`
whitelisted method, In Stock phones only, permlevel-0 fields only; "Search by
Brand/Model" dialog on the form; row click fills IMEI and phone_type confirmed
manually in browser as both Staff and Admin), Customer `address` field
(optional Small Text, migrated and confirmed in `tabCustomer` schema),
`process_sale()` server-side `phone_type` fallback (commit 36c2abf — backfills
`phone_type` from the linked Phone doc if the client-side JS didn't set it;
confirmed 2026-07-21 with a real test submit via console: created a Sales
Entry with `phone_type` unset, submitted it, confirmed it was correctly
backfilled to "Used" with correct margin/VAT/net_profit, then cleaned up
— test entry cancelled+deleted, phone reverted to In Stock, no residue).

**Next task**: none currently queued — small unfinished items are all closed
out.

**Small unfinished items**: none currently open.

**Multi-category expansion (accessories) — Phase 1 built, two real bugs
found and fixed via actual Staff browser testing** (commits 131356d,
edfda3b, 2026-07-21): reuses ERPNext's native `Item`/`Item Barcode` as
product/barcode master data (no Stock Ledger/Warehouse/accounting
involvement, same reuse-not-duplicate pattern as Supplier); new
`current_stock` Custom Field on `Item` (Int, read-only, default 0) via
`fixtures/custom_field.json`, scoped by filter in `hooks.py` (not a bare
doctype export); new `Item Purchase` doctype (mirrors `Purchase Entry`:
item_code/item_name(fetched)/qty/supplier/purchase_price/purchase_date,
naming series `IP-.YYYY.-.MM.-.#####`) whose `on_submit` atomically
increments `Item.current_stock` via raw SQL; whitelisted
`get_item_by_barcode()` looks up the `Item Barcode` child table;
`imei_scanner.js` generalized (`addScanButton`/`openScanner` now take an
optional `options` object — fieldname/buttonLabel/dialogTitle/
successMessage/onScanSuccess callback) so `Item Purchase` reuses it for
barcode scanning without touching the existing Sales Entry/Purchase Entry
callers, which pass no options and are unaffected.

Bugs caught: (1) `validate()`'s `if self.qty and self.qty <= 0` silently
skipped the check when `qty == 0` (falsy-zero) — caught in my own console
testing, fixed to `if self.qty is not None and self.qty <= 0`. (2) Staff
had zero permission rows on core `Item` at all, so the barcode-scan's
`fetch_from` on `item_name` failed with "Cannot Fetch Values" — caught by
the user's own Staff browser test, fixed with a `Custom DocPerm` granting
Staff Read-only on Item (Role-Permission-Manager style, same pattern as
the original Supplier grant, but this one IS persisted as a fixture,
unlike that one). (3) `purchase_price` was wrongly `permlevel 1` on Item
Purchase (copied from `Phone`'s cost field instead of `Purchase Entry`'s,
which is `permlevel 0`) — per explicit decision, staff enter this value
themselves so it isn't sensitive like margin/net_profit; removed. Confirmed
via live meta check that `Purchase Entry.purchase_price` never had this
problem. `margin`/`net_profit` on `Sales Entry` remain `permlevel 1`,
untouched.

**Phase 1 fully verified 2026-07-21**: user confirmed in browser as both
Mobile Shop Staff and Admin — good on both roles. Phase 1 is closed out.

**Phase 2 — unified `Shop Sale` checkout, built and functionally verified,
not yet browser-verified** (commit f74c02d, 2026-07-21): new `Shop Sale`
doctype with two child tables, `Phone Sale Item` (IMEI) and `Item Sale
Line` (barcode/qty), so one sale can mix phones and accessories. Extracted
the New-phone Inclusive/Exclusive and Used-phone PMS formulas out of
`sales_entry.py` into shared `mobile_shop/utils/vat.py`; re-ran the NBR
regression case after the refactor (2000→3000 = margin 1000/vat 90.909/
net 909.091) — unchanged. `validate()` hard-blocks a Used/PMS phone line
from coexisting with any standard-VAT line (New phone OR accessory) — a
New phone shows its VAT amount explicitly, same conflict as PMS+accessory.

Bug caught during review (by the user, before accepting Phase 2 as done):
the first version of that check only looked at `item_sale_lines`, so a
Used+New phone pair (zero accessories) slipped through undetected — fixed
by widening the check to cover any PMS-vs-standard combination, not just
phone-vs-accessory. All five combinations re-verified directly against
`insert()`: Used+New blocked, Used+Accessory blocked, New+Accessory
allowed, Used+Used allowed, New+New allowed.

Before trusting `permlevel 1` on child table fields — new territory, this
app had no child tables before, and margin/vat_amount/net_profit hiding is
compliance-critical — traced Frappe's source
(`apply_fieldlevel_read_permissions` in `document.py`) to confirm child-row
permlevel fields are redacted based on the *parent* doctype's permission
rows, then verified empirically via the real `getdoc` code path (what the
desk form actually uses) as both the live Staff user and Administrator:
Staff cannot see `margin`/`vat_amount`/`net_profit` on phone rows or
`total_vat_amount` on the parent; Admin sees everything. Also ran the full
create/save/submit flow as the real Staff user. All test data cleaned up
each round, confirmed no residue, pre-existing user data untouched.

**Phase 2 fully verified 2026-07-21**: user confirmed in browser as both
Mobile Shop Staff and Admin — good on both roles. Phase 2 is closed out.

**Phase 3a — Shop Sale print formats, built and functionally verified, not
yet browser-verified** (commit 9d810e8, 2026-07-21): two new formats,
`Shop Sale PMS Invoice` (Used-phone sales, never shows VAT) and
`Shop Sale Standard Invoice` (New-phone/accessory sales, itemizes every
line then Subtotal/VAT Amount/Total Charged via the same
`frappe.db.get_value` bypass pattern as the existing Standard VAT Invoice).
Same caveat as the existing two invoices: Frappe defaults to the
auto-generated print unless the correct format is explicitly picked from
the dropdown.

While building this, found that `Print Format` was never listed in
`hooks.py`'s `fixtures` — the 3 pre-existing invoice formats have only
ever lived in the site DB since 2024, `fixtures/print_format.json` was a
disconnected snapshot `bench migrate` never synced. Fixed (scoped fixture
filter, same pattern as the `Custom Field`/`Custom DocPerm` fixes);
verified via two migrates that this is insert-only for the 3 existing
records — their content untouched, sync now real going forward.

Bug caught in review before commit (user, not yet a browser test): the
phone brand/model line only guarded against no Phone being found, not
against an existing Phone having a blank `model` — rendered the literal
word "None" (e.g. "Tecno None"). Same block was copy-pasted into both new
formats. Fixed to `{{ phone.brand or '' }} {{ phone.model or '' }}` in
both. Verified via the real print pipeline (`frappe.get_print`) as both
Staff and Administrator against real submitted `Shop Sale` docs — correct
VAT display per format/role, and confirmed the blank-model case no longer
shows "None".

**Still pending**: a human browser pass on Phase 3a (both formats, both
roles).

**Customer/ERPNext-core naming collision (Hard Rule 10) — explicitly deferred
2026-07-21.** Options considered: rename mobile_shop's doctype (cleanest,
frees up "Customer" for native ERPNext use later, but touches Link field
options across Sales Entry/reports/print formats/workspace); formally take
over "Customer" and drop the ~50 dead ERPNext-core columns via a patch
(simpler, but commits to never using ERPNext's native Customer/Sales Invoice
without redoing this); leave as-is (chosen — not causing runtime problems
today, revisit only if/when accounting integration is actually picked up).
Do not silently "fix" this later without re-raising it — it was a deliberate
deferral, not an oversight.

**Do not start without explicit confirmation** (open questions, decisions pending):
multi-category expansion (earbuds/accessories) Phase 2 onward — Phase 1
(Item/Item Barcode reuse + Item Purchase stock-in) is done, see "Current
state" above; Phase 2 (unified `Shop Sale` checkout mixing Phone + Item
lines) and Phase 3 (retiring Sales Entry, updating reports) are still
gated, full accounting integration
(migrating custom Sales Entry/Purchase Entry to post to ERPNext's native
Sales Invoice/Purchase Invoice/GL — a real architecture decision, not a bolt-on;
**blocked on resolving the Customer/ERPNext-core naming collision first, see
Hard Rule 10** — native Sales Invoice depends on ERPNext's own Customer
doctype behavior), any change to whether PMS invoices show a VAT amount
(would contradict the NBR guide's explicit "must not show VAT amount" rule —
needs reconfirmation, not assumption). Also pending a decision (not urgent
until the above is picked up): how to resolve the Customer naming collision
itself — rename mobile_shop's doctype vs. formally cleaning up/taking over
ERPNext's Customer.

## Hard rules — these came from real bugs, don't relearn them the hard way

1. **Workspace and Number Card are "standard" doctypes.** They sync from
   module-level JSON files on `bench migrate`, but that sync is INSERT-ONLY —
   it never updates an already-existing record, no matter how many times you
   edit the file and migrate. To fix an existing broken record, set values
   directly via `frappe.db.set_value(...)` + `frappe.db.commit()` in the bench
   console. Also: neither doctype should be listed in `hooks.py`'s `fixtures`
   list — that creates a second, competing sync mechanism from
   `fixtures/<doctype>.json` that can silently overwrite real data with stale
   content on every migrate.

2. **Script Report folder/file names are derived automatically from the
   Report's `name` field** (scrubbed: lowercased, spaces→underscores). A
   report named "VAT Report" MUST live in folder `vat_report/` with file
   `vat_report.py` — not a shorter or "cleaner" name. Wrong folder name causes
   `ModuleNotFoundError` at runtime even if the JSON name field is correct.

3. **`"permlevel": 1` on a Script Report column enforces NOTHING by itself.**
   Script Reports run raw SQL directly, bypassing normal document-level field
   permissions entirely. Any sensitive field (purchase_price, margin,
   vat_amount, net_profit) must be omitted ENTIRELY from both the `columns`
   list and the SQL SELECT clause when the user isn't admin — never fetched,
   never a NULL placeholder. Use a shared `has_admin_role()` pattern:
   `bool(set(frappe.get_roles(frappe.session.user)) & {"Mobile Shop Admin", "System Manager"})`.
   For fully Admin-only reports (e.g. Profit Report, VAT Report): the JSON
   `roles` array must list ONLY Admin/System Manager (no Staff), AND
   `execute()` must call `has_admin_role()` as its literal first line and
   `frappe.throw(_("Not permitted"), frappe.PermissionError)` if false —
   belt-and-suspenders, not either/or.

4. **Always verify schema before writing any report/query code.** Run
   `frappe.db.get_table_columns("<Doctype>")` in the bench console and use
   the real output. Never assume field names. A past batch attempt fabricated
   fields like `grade`, `posting_date`, `cr_number` that don't exist anywhere
   in this schema — caused real time loss to find and fix.

5. **Bahrain Dinar uses 3 decimal places (fils)**, not Frappe's Currency
   default of 2. Always set `"precision": "3"` on BHD currency fields.

6. **Naming series need dots around variable tokens**:
   `PE-.YYYY.-.MM.-.#####`, not `PE-YYYY-MM-#####`.

7. **Never run `rm -rf` near `apps/*/public/` or any source directory.**
   Only `sites/assets/` build output is safe to clean. If files do get lost,
   check `git status` / `git log` first — this repo is git-tracked, most
   accidental deletions are recoverable via `git restore`.

8. **When building any query report, build the column list via
   `.append()`/list construction and join with `",\n".join(...)`** — never
   manual string concatenation with commas. A silent double-comma SQL bug
   happened once from hand-managing commas across conditional columns.

9. **Column list and SQL SELECT list must be built together, consistently** —
   don't add a field to one and forget the other; don't leave a stray
   `NULL as fieldname` placeholder when a field is simply supposed to be
   absent for a role.

10. **The custom `Customer` doctype in this app is not cleanly separate from
    ERPNext's core `Customer` doctype — it collided with it and won.** ERPNext
    ships its own `Customer` doctype (`erpnext/selling/doctype/customer/customer.json`,
    module "Selling"). mobile_shop's `customer.json` (module "Mobile Shop") uses
    the identical doctype name. Frappe only allows one `tabDocType` row per
    name, and app doctype-sync is additive-only (never drops columns), so:
    the live `tabDocType` meta for "Customer" is currently mobile_shop's
    version (confirmed via `frappe.get_doc("DocType", "Customer").module ==
    "Mobile Shop"`) because mobile_shop syncs last in `apps.txt` order
    (`frappe, erpnext, mobile_shop`) — but the physical `tabCustomer` MySQL
    table still carries ~50 leftover ERPNext-core columns
    (`customer_type`, `customer_group`, `territory`, `tax_id`,
    `loyalty_program`, etc.) that were never cleaned up, dead weight from
    before mobile_shop's definition took over. Discovered 2026-07-21 while
    adding the `address` field — not caused by that change, pre-existing
    since Customer was first created. This is the same failure mode the
    Supplier decision deliberately avoided (see "General conventions"
    below), just triggered in the opposite direction. Implication: **any
    future work that touches ERPNext's native Customer-linked features
    (Sales Invoice, Quotation, core Selling reports, the accounting-integration
    item in "Do not start without explicit confirmation") must account for
    this collision first** — don't assume "Customer" behaves like a normal
    ERPNext core doctype, and don't assume it's a clean standalone custom
    doctype either. Not yet fixed; needs a real decision (rename mobile_shop's
    doctype, or formally take over/clean up ERPNext's Customer) before it's
    touched further.

## Verification discipline — do not skip this

After any migrate, DO NOT assume a fix worked just because the command
succeeded with no errors. For Workspace/Number Card/Report changes
specifically, run `bench migrate` at least twice in a row and check the
actual browser state each time — one successful migrate is not proof a fix
is permanent (see rule 1).

After implementing anything with role-based visibility (permlevel fields,
Admin-only reports), the verification is NOT complete until a human has
manually checked BOTH roles in the actual browser — Staff should see the
restricted view, Admin should see everything, and for hard-blocked reports,
Staff should get a genuine permission error, not a blank page. Do not report
something as "verified" based on code review or a dry run alone.

## General conventions (from AI_RULES.md)

- Never modify ERPNext core. All customizations stay inside the `mobile_shop` app.
- Reuse ERPNext features whenever possible; never duplicate existing ERPNext
  functionality. (This is why Supplier uses ERPNext's core doctype rather than
  a custom one — a past attempt at a custom Supplier caused a naming collision
  that silently broke migration. Apply the same instinct to the pending
  multi-category/Item decision.)
- Prefer configuration over hardcoded values.
- Python for business logic; JavaScript only for UI enhancements.
- All VAT/margin calculations run server-side only — never trust client-side
  calculations. Validate all user input. Validate IMEI uniqueness before save.
- Mobile-first, responsive, minimal typing, large touch targets. (This is a
  live open item — see "Open items" above regarding custom UI / POS.)
- Type hints where appropriate; clear comments for complex logic; small
  functions; reuse utilities (e.g. the shared `has_admin_role()` pattern,
  the shared `mobile_shop/utils/imei.py` validation module).
- **When unsure, stop and ask — do not guess requirements or introduce new
  dependencies.** This has been the single most effective rule in practice:
  every real bug in this project was ultimately caught by stopping to verify
  (schema, browser state, actual file contents) rather than assuming.

### One important update to "one feature per prompt, finish before starting another"

This rule is correct and validated by real experience — the one time it was
violated (batch-implementing 4 reports at once without individual review)
produced reports built against a completely fabricated schema, which had to
be deleted and redone one at a time. Hold this rule firmly: **one report/
feature at a time, reviewed and manually verified in the browser before
starting the next**, even when it feels slower.

### Git workflow (from AI_RULES.md, with one addition)

For every completed feature:
1. Review code (see files before trusting a summary of them).
2. Run `bench migrate` and `bench restart`.
3. **For anything touching Workspace, Number Card, or Report doctypes:
   run `bench migrate` a second time and re-check the browser** — these
   are standard doctypes with insert-only sync quirks (see Hard Rule 1)
   and a single successful migrate is not proof a fix is permanent.
4. Test manually in the browser — for anything with role-based visibility,
   test as BOTH Staff and Admin, not just Administrator.
5. Commit with a meaningful message.

## Architecture (from ARCHITECTURE.md, corrected against what was actually built)

**Philosophy**: extend ERPNext, don't replace it. Reuse built-in ERPNext
doctypes wherever possible; keep all custom logic inside `mobile_shop`.

**Built-in ERPNext doctypes reused as-is (never modify directly)**: Supplier,
User, Role, Address, Contact, Print Format, Report. (Note: Customer was
*intended* as a fully separate custom doctype, not ERPNext's core Customer —
but it actually collides with ERPNext's core Customer doctype of the same
name and currently overrides it. See Hard Rule 10 before assuming either
description is accurate.)

**Custom doctypes**: Phone, Customer, Purchase Entry, Sales Entry, plus 7
Report doctypes (IMEI History, Sales, Purchase, Customer, Supplier,
Inventory, Profit — VAT Report pending), 3 Number Cards, 1 Workspace.

**"Mobile Shop Settings" singleton (VAT Rate, Business Name, Invoice Footer,
Warranty Defaults) — NOT YET BUILT.** VAT rates are currently hardcoded in
calculation logic, not read from a settings doctype. Legitimate future
improvement, but do not assume this doctype exists.

### Corrected folder structure (the real, working structure)

The original architecture doc showed `mobile_shop/doctype/...` — this is
WRONG and caused a real bug once (a doctype placed at that level silently
failed to register in migrate). The actual, verified-working structure
nests one level deeper:

```
mobile_shop/                           <- app root
├── mobile_shop/                       <- module folder (note: same name, nested)
│   ├── mobile_shop/                   <- doctype module (name matches modules.txt)
│   │   ├── doctype/
│   │   │   ├── phone/
│   │   │   ├── customer/
│   │   │   ├── purchase_entry/
│   │   │   └── sales_entry/
│   │   ├── report/
│   │   │   ├── imei_history_report/
│   │   │   ├── sales_report/
│   │   │   ├── purchase_report/
│   │   │   ├── customer_report/
│   │   │   ├── supplier_report/
│   │   │   ├── inventory_report/
│   │   │   ├── profit_report/
│   │   │   └── vat_report/            <- pending
│   │   ├── number_card/
│   │   │   ├── phones_in_stock/
│   │   │   ├── sales_this_month/
│   │   │   └── purchases_this_month/
│   │   ├── workspace/
│   │   │   └── mobile_shop/
│   │   └── utils/
│   │       └── imei.py                <- shared IMEI validation, used by
│   │                                      Phone, Purchase Entry, Sales Entry
│   ├── public/js/
│   │   ├── imei_scanner.js            <- shared camera scanner module
│   │   ├── sales_entry.js
│   │   └── purchase_entry.js
│   └── hooks.py
```

Always verify against this real structure (or `find mobile_shop/mobile_shop
-maxdepth 3 -type d` on the actual repo) before creating a new doctype/
report/etc. — never assume a shallower path.

### Relationships

```
Supplier (ERPNext core) --> Purchase Entry --> Phone --> Sales Entry --> Customer (custom)
```

### Bahrain VAT — current, correct logic (supersedes the single-formula
version in the original doc, which predates the New/Used split)

**Used phones (Profit Margin Scheme)**:
```
Margin = Selling Price - Purchase Price
If Margin <= 0: Margin = 0, VAT = 0, Net Profit = 0
Else: VAT = Margin / 11
      Net Profit = Margin / 1.1
```
Verified against the actual NBR Profit Margin Scheme guide. Invoice must NOT
show the VAT amount (NBR requirement) — only that VAT is included under PMS.

**New phones, Inclusive**:
```
VAT = Selling Price / 11
Net Profit = Selling Price / 1.1
Total Charged = Selling Price
```

**New phones, Exclusive**:
```
VAT = Selling Price × 0.10
Net Profit = Selling Price
Total Charged = Selling Price × 1.10
```
New-phone invoices (Standard VAT Invoice) DO show the VAT amount explicitly
— opposite of the PMS invoice rule above.

`margin`, `vat_amount`, `net_profit`, `purchase_price` are permlevel-1
fields — see Hard Rule 3 above for how visibility is actually enforced
(never just the permlevel tag alone).

### Future expansion (from ARCHITECTURE.md — now cross-referenced against
actual client requests, see "Open items" above)

The original doc already anticipated: ERPNext Stock integration, Serial
Number integration, Purchase Receipt integration, Sales Invoice integration,
repair tracking, multi-branch, warranty. This turned out to correctly
anticipate the client's later accounting-integration proposal (Phase A) and
the accountant's bulk-intake-by-model / IMEI-at-sale-only workflow — i.e.
the direction this was always heading is now confirmed as the right one,
just not yet started. See "Open items" above for current status of each.

## Environment

- Bench root: `~/Documents/Work/mobile_shop/frappe-bench`
- App path: `~/Documents/Work/mobile_shop/frappe-bench/apps/mobile_shop`
- Site: `mobileshop.local`
- Start dev server: `bench start` (must be running in its own terminal)
- Migrate: `bench --site mobileshop.local migrate`
- Console: `bench --site mobileshop.local console`
- Two git repos exist: this inner one (`apps/mobile_shop/`) is the one that
  matters and has real commit history. An outer repo at
  `~/Documents/Work/mobile_shop/` is mostly unused — don't worry about it.
- Camera-scanning features require HTTPS or localhost (browser restriction).
  Use an ngrok tunnel for testing on a real device:
  `ngrok http 8000 --host-header="mobileshop.local:8000"`
