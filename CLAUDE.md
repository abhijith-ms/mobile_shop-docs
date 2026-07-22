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
in browser). **Correction 2026-07-22**: "8 of 8 on the dashboard" was never actually
true — the reports themselves were built and permission-verified correctly, but the
live Workspace record was missing shortcuts for Inventory/Profit/VAT Report entirely
(see the workspace-drift entry further down); fixed as part of the POS work.
Sales Entry search by brand/model (committed 57e6ea7 — `search_phones()`
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

**Next task**: three narrow items left on the POS, all human/hardware-side,
no code known to be needed — (1) the camera-scanner path specifically on
a real tablet/phone browser (desktop testing so far used typed/wedge
input); (2) a quick pass as the real Staff user confirming no margin/VAT/
profit figures appear anywhere in the POS UI; (3) one real print once the
thermal printer physically arrives, to confirm the driver honors 80mm
sizing (stated from the start as the one thing that can't be verified
without hardware). Once those land: nothing else currently queued for the
multi-category expansion plan — Phases 1, 2, 3a (code), and 3b are all
done. One thing still outstanding, human-side,
not code: decide whether `Sales Entry` stays visible in the workspace nav as a
historical-only doctype or gets hidden for new-entry purposes (explicitly
deferred design question, not urgent). Full plan at
`~/.claude/plans/mossy-brewing-wren.md`.

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

**Phase 3b — reviewed report-by-report plan approved 2026-07-21, in
progress.** Reading the actual code first (not assuming) found only 5 of
the 8 reports touch `Sales Entry` at all — Purchase Report, Supplier
Report, Inventory Report query only `Purchase Entry`/`Phone`/`Supplier`
and are staying untouched, as approved. Order: IMEI History → Customer →
Sales → Profit → VAT (simplest/lowest-stakes first, Admin-only
compliance-critical last), then the `Sales This Month` Number Card
(document_type swap, not a report change) once the pattern is proven.
Each report: snapshot real baseline before touching anything, implement,
confirm historical rows byte-identical, fresh create-sell-verify-cleanup-
reconfirm cycle, commit individually (clean revert point per report, not
one blob).

1. **IMEI History Report — done** (commit 4f143e7). LEFT JOIN + COALESCE
   (not UNION — this report is Phone-centric, one row per Phone) onto
   `Phone Sale Item`/`Shop Sale`, both filtered `docstatus=1`.
   `invoice_number` column changed `Link(Sales Entry)` → `Data` since it
   can now hold a name from either doctype. Two real gaps caught by the
   user reviewing before moving to report #2 (worth remembering for any
   future report touching multiple sale sources): (a) confirmed
   empirically, not just by reading the SQL, that the `docstatus=1` join
   filters actually work — cancelling a Shop Sale correctly drops its row
   with no duplication; (b) a phone with a submitted row in BOTH sources
   (possible via the unsupported manual-status-reset "return" path) was
   resolved by blind `COALESCE(se.x, ss.x)` — Sales Entry always won
   regardless of which was actually more recent. Fixed to compare
   `creation` timestamps and pick one source consistently across all four
   sale-related columns together (never mixed field-by-field). Also
   discovered 3 of the user's own real phones (sold via Shop Sale during
   Phase 2/3a testing) were showing `Sold` with completely blank sale info
   under the old report — confirmed all 3 now correct.

2. **Customer Report — done** (commit ccb2645). `purchase_count`/
   `total_sales` count per Shop Sale *document*, not per line item —
   explicit design decision confirmed with the user before implementing:
   reuses Shop Sale's own already-computed `total_charged` (UNION ALL of
   Sales Entry + Shop Sale, both `docstatus=1`, then grouped) rather than
   reconstructing the Inclusive/Exclusive/PMS VAT math per line in raw SQL
   a second time. Verified the historical-only customer byte-identical,
   independently cross-checked the user's real customer's new totals (4
   purchases, 1400.000) against a manual `frappe.get_all` sum rather than
   trusting the report's own SQL. Noted but explicitly untouched: two of
   the user's real historical Sales Entry records have `total_charged =
   0.000` despite a non-zero `selling_price` — pre-existing data quality
   issue, unrelated to this work.

3. **Sales Report — done** (commit a551a5f). Most complex of the 5: UNION
   ALL of Sales Entry + `Phone Sale Item` + `Item Sale Line`, `docstatus
   IN (0,1)` throughout (matches this report's existing draft-inclusive
   behavior), filters applied once on the outer query against unified
   column names rather than duplicated per branch. New "Line Type"
   (Phone/Accessory) column, item_name in the Model/Description column
   for accessory rows, IMEI/Brand blank — per the design decision made
   before implementing. Brand/Model filters exclude accessory lines;
   surfaced via the report's `message` return value (shown as a note atop
   the report) whenever either filter is active, so a "Samsung" filter's
   total can't be misread as "all Samsung revenue." Caught a bug in my own
   first draft before testing: the Model filter matched the unified
   `model` output column, which doubles as item_name for accessories —
   without a `line_type='Phone'` guard, a model search could have
   incidentally matched an accessory whose name coincided with a phone
   model string. `total_charged` per line computed inline from
   selling_price/phone_type/vat_treatment only (never from the
   permlevel-1 fields) since it's permlevel 0 everywhere already and the
   formula is the same non-sensitive relationship already in
   `mobile_shop/utils/vat.py`.

   Security-critical per explicit instruction (this report is
   Staff-visible and now touches `Item Sale Line.vat_amount`, permlevel 1,
   new in this report): purchase_price/margin/vat_amount/net_profit
   appended to every UNION branch only inside the `is_admin` guard, never
   a NULL placeholder. Verified as the real Staff user, not just by
   inspecting the column list — confirmed all four fields absent from
   both the column list and every row's actual data across all 9 real
   rows. Also independently cross-checked Shop Sale row numbers against
   the real document (a mixed phone+accessory sale's two report rows sum
   to the document's own `total_charged`), and specifically tested the
   Exclusive-VAT branch (no existing real data covered it) with a fresh
   phone+accessory Shop Sale — both lines' computed totals reconciled
   exactly with the real submitted document.

4. **Profit Report — done** (commit 760954a). Line-level, not
   document-level — distinct from Customer Report's per-document choice,
   per explicit instruction: UNION ALL with ONLY `Phone Sale Item`, never
   `Item Sale Line`. Accessories carry no margin/profit concept, so an
   accessory-only sale contributes nothing and a mixed New-phone+accessory
   sale contributes only its phone line, by construction (Item Sale Line
   is structurally absent from the query) rather than by a filter that
   could be gotten wrong. `has_admin_role()` hard gate confirmed unchanged
   (still the literal first line of `execute()`) and re-verified as the
   real Staff user — hard-blocked with `PermissionError`.

   Verified: historical rows byte-identical; confirmed the real
   accessory-only Shop Sale is entirely absent from the report; explicit
   test per instruction - a fresh mixed New-phone+accessory Shop Sale
   (document `total_charged` 280.000 = 220 phone + 60 accessory) produces
   exactly one report row showing 220.000, not 280.000, confirming
   accessory revenue never reaches profit math even when bundled with a
   phone in the same document.

5. **VAT Report — done** (commit 46b033c). Most compliance-critical of
   the five. UNION ALL of Sales Entry + `Phone Sale Item` (PMS/Standard by
   `phone_type`) + `Item Sale Line` (always Standard). Detail rows sum
   `vat_amount` at LINE level; summary card transaction counts use
   PER-DOCUMENT counting instead (explicit decision, distinct from Profit
   Report's line-level choice) — unambiguous specifically because
   `validate_no_pms_standard_mix` guarantees every Shop Sale document
   belongs to exactly one scheme, so a document's rows never split
   between the two count buckets. `has_admin_role()` gate confirmed
   unchanged.

   Strictest verification of all five: recorded exact Standard/PMS/
   grand-total VAT for 3 date ranges (all-time, the one real month, a
   zero-data month) from both the live report AND independent raw SQL
   before touching anything — PMS 27.272/2 transactions, matched exactly
   both ways. After the change: PMS side exact match on all 3 ranges
   (untouched, no historical PMS Shop Sale data exists); Standard side
   (all new) independently cross-checked via raw SQL summing both child
   tables' vat_amount and counting distinct documents separately —
   127.273 across 4 documents, matched exactly. Fresh isolated-date-range
   test (2026-03-15) with one Shop Sale per scheme, including a mixed
   New-phone+accessory document as the direct test of the counting
   decision — 4 detail rows across 3 documents, confirmed the mixed
   document counted as exactly 1 Standard transaction not 2, every
   figure matched hand-computed expectations exactly. Cleaned up,
   reconfirmed all 3 historical ranges (including the isolated range
   dropping back to zero) exactly unchanged.

**Phase 3b fully done 2026-07-21** (commit e7e1527 closes it out): all 5
reports needing Shop Sale integration done (Purchase Report, Supplier
Report, Inventory Report correctly left untouched, never touched Sales
Entry) plus the `Sales This Month` Number Card — `document_type` swapped
Sales Entry → Shop Sale via direct `db.set_value` (Hard Rule 1, module
JSON alone would never have touched the live record), explicit
`docstatus=1` filter added. Worth remembering: the OLD card never
actually had a docstatus filter either — `frappe.get_list()` doesn't
implicitly exclude drafts/cancelled, confirmed empirically; the dashboard
happened to look right only because no draft/cancelled Sales Entries
existed this month, not because anything filtered them. The new card's
`docstatus=1` filter is a deliberate correction, not a faithful carry-over
of what the old one technically did. Verified via the real `get_result()`
call the dashboard actually invokes: matched an independent manual COUNT,
confirmed a fresh draft+cancelled pair didn't move the count, confirmed a
fresh submitted one incremented by exactly 1.

**Multi-category expansion (accessories) plan is now fully implemented in
code** (Phases 1, 2, 3a, 3b).

**Custom POS checkout screen — built in 4 commits, desktop end-to-end path
now fully human-verified 2026-07-22** (plan at
`~/.claude/plans/silly-mapping-piglet.md`). Desk Page at
`/app/mobile-shop-pos` (not a portal page — html5-qrcode/`imei_scanner.js`
load via `app_include_js`, Desk-only). Deliberately thin server layer:
`pos_scan()` resolves a code to a phone or accessory by actual lookup
(never a length heuristic), `create_pos_sale()` builds and submits a real
`Shop Sale` with no `ignore_permissions` — every rule (mixing block, stock
checks, VAT math, permlevel) is inherited from Phase 2's already-verified
code, confirmed empirically (a PMS+accessory cart through the POS is
rejected by the document's own `validate_no_pms_standard_mix`, not a POS-
side reimplementation).

Before building the cart-input flow on it, verified — not assumed from a
code read — that `imei_scanner.js`'s `openScanner(null, {onScanSuccess})`
survives with `frm` null: ran the real file under stubbed browser APIs
driving the whole open→camera-start→scan-success→cleanup path. Camera UI
itself (real DOM/timing) still needs the browser.

`is_walk_in` (Check, on `Customer`) plus a POS-only rule: a cart containing
a phone line requires a named customer, enforced *only* in
`create_pos_sale`, not in `ShopSale.validate()` — confirmed both halves
(blocked via the POS, allowed via a direct document with the same
walk-in+phone combination) and stated back to the user as a deliberate
asymmetry before proceeding: VAT-scheme mixing is an NBR compliance
invariant that must hold everywhere, this is an operational guardrail an
admin correction can legitimately bypass.

Two new 80mm print formats (`Shop Sale PMS Receipt`/`Shop Sale Standard
Receipt`) reuse the exact Jinja from Phase 3a's A4 formats. Sizing
correction worth remembering: goes in Print Format's `css` field, not
`margin_top/bottom/left/right` (PDF-path-only, never reached by classic
`/printview`) — and the `css` has to actively override
`standard.css`'s `.print-format` max-width/padding or the receipt renders
A4-padded on 80mm paper. Print buttons use `frappe.utils.print()` (the
same helper ERPNext's own POS calls), letterhead passed as `''` not
`null` (`encodeURIComponent(null)` → literal string `"null"` in the URL).

A user report of "no print buttons after a sale" turned out not to be a
code bug — reproduced the exact shipped script (fetched via the real
`frappe.desk.desk_page.get` loader) under jsdom + real jQuery with the
exact `.layout-main-section` markup `make_app_page` builds, and it worked
end to end with zero errors. Confirmed live in-browser afterward
(green banner, both buttons, 0 console errors) — left as-is, no fix
needed. Two harness pitfalls hit and fixed along the way, worth
remembering for any future test like this: jQuery 4's UMD wrapper
self-binds at `require()` time if `global.window` is already set (so
`require('jquery')`, not `require('jquery')(window)`); jsdom's
`window.eval` doesn't execute in the window's own realm unless
`runScripts: "dangerously"` is set, so `frappe.provide` must bridge
`window.X`/`global.X` onto the same object for bare identifiers to
resolve.

Workspace shortcut (commit d8c4a4b) surfaced something much bigger than
"add one shortcut": the **live** Workspace record had drifted hard from
the on-disk module JSON (Hard Rule 1 — sync is insert-only) — missing
Inventory/Profit/VAT Report shortcuts entirely (see the corrected "Done
and verified" note above), `module` was `"Setup"` not `"Mobile Shop"`,
`icon` was `"retail"` not `"smartphone"`, `roles` was empty, and `links`
carried 17 unrelated ERPNext-core entries (Chart of Accounts, Item,
Warehouse, etc.) — all consistent with the workspace having been built
via the visual Workspace Editor from a cloned generic template at some
point (matches the editor-history note already in PROJECT_PLAN.md), never
cleaned up. Confirmed the scope expansion with the user before fixing
beyond "add the POS shortcut". Fixed directly on the live record via
`frappe.get_doc(...).save()` (module, icon, links, roles, shortcuts,
content), verified through two migrates AND independently through
`frappe.desk.desktop.get_desktop_page()` — the actual API the browser
calls — confirming all 14 shortcuts resolve cleanly. On-disk file rewritten
to match, using the *live* shortcut structure as the base (Sales
Entry/Purchase Entry as lookup-only List shortcuts) rather than the file's
old "New Sale"/"New Purchase" (`doc_view: New`) design, which predates and
now contradicts the POS decision — staff create sales through the POS, not
the raw Sales Entry form.

**Desktop end-to-end pass, done 2026-07-22**: real mixed New-phone +
accessory sale (a real Item Barcode the user had to debug their way into
creating first — itself confirms the Item Barcode/Item Code distinction
landed correctly, not just that the code happened to work) — scan → cart
→ complete → both Receipt and Invoice printed, independently re-checked
math (250 phone + 400 accessory = 650 subtotal, ×10% Exclusive VAT =
65, total 715 — all correct), 80mm receipt reflow matched the A4 invoice's
figures exactly, and the desk-form Shop Sale document underneath was
structured correctly (one phone line, one accessory line qty 2, correct
rolled-up totals). Desktop path (typed/wedge input) considered solid.

**Small unfinished items**: the 3 items in "Next task" above (tablet
camera-scanner path, Staff-role no-leak pass, one real print once the
thermal printer arrives) — otherwise the POS build is considered done;
human browser verification of Phase 3a's A4 print formats specifically
(separate from the POS's thermal receipts, which have been
live-confirmed).

**Custom homepage/launcher screen — done and browser-verified 2026-07-22**
(plan at `~/.claude/plans/silly-mapping-piglet.md`, substantially revised
mid-build — see below). Replaces the Mobile Shop Workspace as the default
landing surface: tiles for all 3 daily tasks, all 8 reports, and the 3
Records doctypes, filtered per-tile by the same permission gates the
framework itself uses (`Report.is_permitted()` +
`frappe.has_permission(ref_doctype, "report")`, `Page.is_permitted()`,
`frappe.has_permission(doctype, "read")`) — never a new, hand-rolled rule.
Profit Report/VAT Report tiles each check their OWN report module's
`has_admin_role()` (imported separately, never shared) after the user
explicitly caught a plan draft that implied reusing one for both — verified
by confirming `admin_check is profit_report_has_admin_role` (not the VAT
one) for the Profit tile and vice versa.

Found and fixed a real, pre-existing, previously-undocumented bug while
building this (not caused by it): Staff could see the Supplier Report
shortcut on the existing Workspace but got a hard `PermissionError` if they
actually clicked it — confirmed via the real
`frappe.desk.query_report.run()` entrypoint. Root cause: the Supplier
`Custom DocPerm` for Mobile Shop Staff (granted during the Phase 1 Item
work) had `read`/`create`/`export` but never `report`. Fixed by setting
`report: 1` on that row; re-confirmed Staff can now actually run the
report.

The plan's original routing mechanism (`add_to_apps_screen` hook +
`System Settings.default_app`, landing on a standalone Desk Page) turned
out not to work — see Hard Rule 11. Pivoted mid-build, with the user's
explicit approval, to a new "Mobile Shop Home" Workspace (name checked
against collision with ERPNext's own pre-existing "Home" workspace first)
containing a single Custom HTML Block that runs the tile-grid JS/CSS, with
`User.default_workspace` set directly on both real accounts. Also hit and
fixed Hard Rule 12 (the block silently didn't render — nothing to do with
sanitization or shadow-DOM script execution, both of which were
independently ruled out first). The original "Mobile Shop" Workspace is
completely untouched and still reachable from the sidebar.

Verified end-to-end, both roles, in the real browser: correct tiles per
role (Staff: 12, no Profit/VAT; Admin: 14), navigation confirmed for one
tile of each type (DocType/Report/Page), touch-target sizing, and that a
fresh login/`/app` visit lands on the new page while the old Workspace
stays reachable unchanged.

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

11. **`add_to_apps_screen` + `System Settings.default_app`/`User.default_app`
    cannot point at a standalone Desk Page if the app owns any Workspace at
    all.** `frappe.apps.get_default_path()` calls `get_apps()`, which runs
    every hook route through `frappe.apps.get_route()` — and that function
    treats any `/app/<slug>` route as naming a *Workspace*, not a Page. If
    the slug doesn't match a real Workspace name, it silently falls through
    to "the first Workspace belonging to this app's module" and returns
    *that* route instead, discarding the hook's own route value entirely.
    Confirmed empirically (twice) on this app: a hook route of
    `/app/mobile-shop-home` resolved to `/app/mobile-shop` (the existing
    Workspace) for every user. There is no way to point this mechanism at a
    Page while a Workspace exists. To make a non-Workspace-shaped experience
    the default landing surface, either build it as a Workspace (see Hard
    Rule 12 for the Custom HTML Block gotcha) and set `User.default_workspace`
    directly — that field is read *before* `get_default_path()` in
    `auth.py`'s `set_user_info()` and bypasses `get_route()`'s rewriting
    entirely — or don't try to change the default landing page at all.

12. **A Workspace's `content` JSON `custom_block` entry only controls layout
    position — it does NOT make the block available to fetch.** The actual
    data source `get_desktop_page()` (the real API the browser calls) uses
    for `custom_blocks` is a separate child table on the Workspace document
    itself (`custom_blocks`, doctype `Workspace Custom Block`, fields
    `custom_block_name`/`label`) — exactly parallel to how `shortcuts` and
    `number_cards` each need both a `content` block *and* a real child-table
    row, but easy to miss for `custom_block` since most examples only cover
    shortcuts. Omitting the child-table row produces total silence: no
    console error, nothing in the Network tab worth flagging, the block
    element just never gets data and renders nothing — `get_desktop_page()`
    returns `custom_blocks: {"items": []}` even though the Custom HTML Block
    document itself is fine and the `content` JSON correctly names it. Before
    suspecting sanitization or shadow-DOM script execution for a
    non-rendering Custom HTML Block, call `frappe.desk.desktop.get_desktop_page()`
    directly (same technique as the Workspace-drift verification) and check
    whether `custom_blocks.items` is actually populated first.

13. **A Desk Page's client script (`page/<name>/<name>.js`) can be served
    stale from Redis's page-cache even after `bench migrate` + `bench
    restart` + a hard browser refresh (`Ctrl+Shift+R`).** Confirmed while
    editing `mobile_shop_pos.js` to add the customer phone-search button:
    the button was verifiably present in the source file
    (`grep`-confirmed) and `bench build --app mobile_shop` had run clean,
    but the live POS page kept rendering the old markup through repeated
    hard reloads with the browser's own cache cleared each time (console
    even logged "Cleared App Cache"). The fix was
    `bench --site mobileshop.local clear-cache` — only after that did the
    new button appear. This is a distinct failure mode from Hard Rule 1
    (Workspace/Number Card insert-only *doctype* sync): this one is a
    server-side Redis cache of the compiled page bundle, not a database
    sync issue, and no `bench build` output or browser dev-tools signal
    hinted at it. When a verified-correct Page JS edit doesn't show up
    live, reach for `bench clear-cache` before suspecting the browser or
    the edit itself.

14. **The `Administrator` account bypasses every permission check
    unconditionally — testing "as Admin" using it does NOT verify what the
    `Mobile Shop Admin` or `System Manager` *role* actually grants.**
    Discovered while building the POS's void-sale feature: `shop_sale.json`
    had never set a `cancel` permission key for any role, on any of Staff,
    System Manager, or Mobile Shop Admin. Frappe's `cancel` DocPerm field
    defaults to `0` when the key is absent — it does **not** inherit from
    `submit`, confirmed directly in `docperm.json`. So in reality, nobody
    could cancel a Shop Sale. This stayed completely invisible through
    every earlier "Admin" browser-verification pass in this project,
    because every one of them logged in as the literal `Administrator`
    superuser account (per Hard Rule/CLAUDE.md convention, since that's the
    only admin-tier login this project's human tester has used) — and
    `frappe.has_permission()` returns `True` for `Administrator` regardless
    of DocPerm rows entirely. Confirmed empirically:
    `frappe.set_user("Administrator"); frappe.has_permission("Shop Sale",
    "cancel")` → `True`, but the same check against a real non-superuser
    account holding only the `System Manager` role → `False`, on the exact
    same (missing) permission row. **Any future permission-related
    verification that matters — not just UI smoke-testing — needs to run
    against a real non-superuser account carrying the role in question
    (e.g. `abhijithms.9526@gmail.com`, which holds `System Manager`), not
    just `Administrator`.** `Administrator` remains fine for non-permission
    testing (data correctness, UI rendering, business logic) where its
    bypass doesn't matter.

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
