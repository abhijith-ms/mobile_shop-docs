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

**The database is intentionally empty as of 2026-07-25.** A full
test-data wipe (see the dated entry below) cleared every transaction and
master record — Phones, Items, Customers, Suppliers, all sale and
purchase documents — ahead of a full end-to-end test scenario run from
the shared test document. Only `Walk-in Customer` was re-seeded. If a
report, the POS, or the homepage looks broken because nothing is there,
that is expected state, not a regression; restore
`20260725_094524-mobileshop_local-database.sql.gz` if the pre-wipe data
is ever needed back.

**Next task**: (1) A human browser pass on the **Purchase Voucher** work
committed 2026-07-28 (see that section below) — the `+ Add Line` dialog,
the per-row IMEI capture dialog with its live "5 / 10" counter and camera
button, the per-line VAT Treatment columns in both grids, a real mixed
voucher submit, and a real cancel with its block message rendered as an
actual dialog. Also (2) a human pass on the intake-cancel work committed
2026-07-25 — Cancel present for Admin and absent for Staff on all three
older intake doctypes. Then two hardware-side POS items, no code known to
be needed — (3) the camera-scanner path on a real tablet/phone browser
(desktop testing so far used typed/wedge input); (4) one real print once
the thermal printer arrives, to confirm 80mm sizing (stated from the start
as the one thing that can't be verified without hardware).

Two questions are open **for the accountant**, both recorded in the
Purchase Voucher section: what `Phone.purchase_price` should store on a
VAT-exclusive line (currently stored exactly as entered), and whether a
voucher mixing Used and standard-VAT lines should harden from a warning
into a hard block.

**A full accountant meeting happened 2026-09-02** (separate from the two
open questions above, which are still unanswered) — full detail in
PROJECT_PLAN.md's "Accounting & POS Feature Set — Phase Plan" section: ten
feature requests plus follow-up answers, a settled bespoke/GL-ready
accounting architecture decision (Hard Rule 20), and a Phase 1–6 build
sequence. See PROJECT_PLAN.md for items 1c (live POS VAT display) and 1d
(recent sales in POS), each still gated behind its own plan-before-code
cycle once the items ahead of it are browser-verified.

**1a (Purchase Report VAT columns, commit `21839d6`) — code and API-level
verification now done, real browser click-through still outstanding.**
Re-verified 2026-09-02: both real non-superuser accounts
(`clashams4@gmail.com` / Mobile Shop Staff, `msadmin.test@mobileshop.local`
/ Mobile Shop Admin — never `Administrator`) get identical columns
including `purchase_price`/`net_amount`/`vat_amount`/`amount` via the real
`frappe.desk.query_report.run()` entrypoint. The legacy-source blank-vs-zero
case (flagged as untested in the original commit, since zero real Purchase
Entry/Phone Batch Purchase rows existed to check) was closed by creating
one of each as real submitted documents (manifest-tracked per Hard Rule
18), confirming `net_amount`/`vat_amount` arrive as JSON `null` (not `0.0`)
in that same API response — the exact payload the browser's report
DataTable renders from, and null already confirmed to render blank rather
than `0.000` when this identical pattern shipped on Accessory Purchase
Report. Both test documents were then cancelled (as the real Admin
account — cancel is admin-only) and deleted, including the side-effect
`Phone` (auto-deleted by `Purchase Entry.on_cancel`) and the synthetic
`Phone Batch` master (deleted directly once its `untracked_qty` confirmed
back to 0); zero residue confirmed, the 3 real Purchase Vouchers and the 1
real Phone Batch (`8534578896`) confirmed unchanged throughout.

**What's still missing**: an actual browser screenshot/click-through as
both roles. No Claude-in-Chrome connection was available this session
either — same gap the original commit disclosed. This is a real, not
cosmetic, gap: the API check proves the *data* reaching the frontend is
correct, not that the DataTable actually renders it correctly on screen.
Needs a human (or a future session with a working browser connection) to
open the report as both accounts and eyeball it before 1a is fully closed.

**1b (suggested sale price at intake) — planned, not yet reviewed or
built.** Full plan at `~/.claude/plans/suggested-sale-price-at-intake.md`:
new `suggested_sale_price` field on `Phone`, `Purchase Voucher Phone Line`,
and `Phone Batch` (mirrors `last_purchase_price`'s overwrite-on-restock
pattern for the batch case); flows through `create_phone_record()`,
`add_to_phone_batch()`, and `create_phone_from_batch()` exactly like
`purchase_price` already does; pre-fills the POS cart's price input via a
`pos_scan()` field addition, replacing today's hardcoded `selling_price: 0`,
while staying fully editable at the till. **Deliberately permlevel 0
everywhere, diverging from `Phone.purchase_price`'s permlevel 1** — reasoned
out explicitly in the plan (not margin-sensitive, Staff already see and
type the number at both intake and the till). Deliberately does NOT touch
`Purchase Entry`/`Item Purchase`/`Phone Batch Purchase` — those three are
historical-only intake forms with no homepage tile, so Phones created
through them will simply have a permanently blank suggested price, the same
NULL-not-zero honesty used elsewhere. Needs review before any code is
written.

**Purchase Report's blank Model/Storage columns — investigated
2026-09-02, not a bug, needs a scoping decision.** Every live row in
Purchase Report is a Purchase Voucher row right now (there are currently
zero submitted Purchase Entry/Phone Batch Purchase documents — all real
intake has moved to Purchase Voucher), and Purchase Voucher Phone Line has
no separate Model/Storage fields at all — by the client's own explicit
request, brand and model are folded into one free-text `brand` field (this
is the deliberate design already recorded in the Purchase Voucher section
below, and already flagged as an open item: "Whether `Phone.model` staying
blank is acceptable"). Confirmed against real data: `Purchase Voucher Phone
Line.brand` holds values like `"iPhone 16 Pro"`, `"Samsung S25 Ultra"`,
`"iPhone 17"` — exactly what showed up in the Brand column with Model/
Storage blank next to them. Purchase Report's SQL already explicitly sets
`NULL AS model, NULL AS storage` for the Purchase Voucher branch (commented
in `purchase_report.py`), while the Purchase Entry and Phone Batch Purchase
branches correctly select real `model`/`storage` values — confirmed via the
1a test data above (both test records were entered with distinct
model/storage values and read back correctly by the report's SQL for those
two branches). So nothing is miswired and nothing needs fixing in the
report itself; the report is correctly surfacing a pre-existing intake gap.
The open decision (unchanged from when it was first flagged, just now
visibly affecting Purchase Report too): leave it as-is, or have
Purchase/Sales/Inventory Report's Model column and filter fall back to
parsing/searching the free-text `brand` field. Not decided; do not "fix"
Purchase Report unilaterally.

The report follow-up phase that used to sit here is **done** (2026-07-29,
see below): `Purchase Report` and `Supplier Report` now read every
purchase source, and the accessory gap that phase surfaced was closed by
the new `Accessory Purchase Report`. Every purchase source is now
reported.

The old "does `Sales Entry` stay in the workspace nav" question is now
**answered** (2026-07-28, then superseded 2026-07-29): it, `Purchase
Entry`, `Item Purchase` and `Phone Batch Purchase` first moved to a
"Historical" section on the homepage launcher, and that section was then
**removed entirely** (commit `f444ca6`) — they have **no homepage tile at
all** now, for either role. Purchase Voucher and Shop Sale cover every
new-entry case, and the multi-source reports already surface the older
documents' data. Full plans at `~/.claude/plans/mossy-brewing-wren.md`
and `~/.claude/plans/new-feature-scoping-for-immutable-puppy.md`.

**Nothing about access moved, and that distinction is the point** — all
four keep every permission row, stay searchable, stay openable from their
List views (`+ Add` included), and stay cancellable by an admin. They
have to: `Purchase Report`/`Supplier Report` read `Purchase Entry`
directly, and `Purchase Entry.phone_created` is the only real `Link` to
`Phone` in the schema. A comment sits where the section was, saying both
"do not restore these tiles" and "do not follow this through into the
permission rows".

**Worth remembering — Hard Rule 1 does NOT apply to `home_tiles.py`.**
The instinct to reach for `frappe.db.set_value()` + two migrates was
right for the Workspace and the Custom HTML Block and wrong here.
`TILE_SECTIONS` is a plain Python list in a module; the launcher block
calls `get_homepage_tiles()` at runtime and hardcodes **no** tile labels
or section names, so there is no DB row to drift and nothing for
`set_value` to target. Editing the Python and restarting is the whole
fix. Confirmed by reading the live block's script, not assumed. The
Workspace and Custom HTML Block records needed no change at all, because
neither one names a section.

**`Accessory Purchase Report` — built and browser-verified 2026-07-29**
(commit `8077b0d`). The counterpart to Purchase Report, closing the gap
that report's phones-only scope deliberately left. Unions `Item Purchase`
and `Purchase Voucher Accessory Line`; see the coverage table in the
Architecture section for how the two reports partition the sources.

**The VAT asymmetry is represented honestly rather than flattened.**
`Item Purchase` predates per-line VAT entirely, so its `vat_treatment`,
`net_amount` and `vat_amount` are **NULL, not zero** — zero would assert
"no VAT was charged", which that form never recorded either way. Its
`amount` IS populated, because `qty * price` is genuinely known. A
message above the report explains the blanks so they read as a fact
about the older form, not as missing data.

**The permlevel inconsistency this surfaced is now RESOLVED** (commit
`2037845`, same day): all three purchase-side reports — Purchase,
Accessory Purchase and Supplier — show their money columns to Staff,
matching the permlevel 0 of every field they read. They had been stricter
than their own doctypes for months, which protected nothing since Staff
type those numbers into the intake forms. Sales Report, Profit Report and
VAT Report are untouched; see Hard Rule 3's table for which fields are
genuinely sensitive and why `purchase_price` appears on both sides of it.

**Purchase Report + Supplier Report extended to every purchase source —
done and browser-verified 2026-07-29** (3 commits: `c3be0c9`, `3c27969`,
`4708147`; plan at
`~/.claude/plans/purchase-and-supplier-report-multi-source.md`). Closes
the gap flagged when Purchase Voucher shipped. The audit found it was
wider than expected: **neither report had ever included `Item Purchase`
or `Phone Batch Purchase`**, so those gaps long predate Purchase
Voucher.

**Two deliberate asymmetries between the two reports — do NOT "fix"
either to match the other.** Both are commented in their own files:

- **Purchase Report is phones only.** It UNIONs `Purchase Entry`,
  `Phone Batch Purchase` and `Purchase Voucher Phone Line`, and
  *structurally omits* `Item Purchase` and `Purchase Voucher Accessory
  Line` — the same technique Profit Report uses to keep `Item Sale Line`
  out, so it cannot be got wrong later by editing a filter. Its columns
  are phone-shaped (IMEI/Brand/Model/Storage/Phone Type) and an accessory
  row would be five blank columns. A `message` above the report says so.
  **Consequence: accessory purchases appear in NO report today** — that
  is a known, accepted gap, and belongs in its own report if wanted.
- **Supplier Report includes everything**, accessories included, because
  it answers "what have we bought from this supplier" and a distributor's
  earphone spend is part of that.

**Counting rule that makes the aggregates correct:** Supplier Report
pre-aggregates each source to **document grain** in a `UNION ALL`
subquery *before* joining `Supplier`. Joining straight to a voucher's
child tables fans out — one document with three lines would count as
three purchases and have its value summed three times. Verified against
ground truth computed independently with `frappe.get_all` (not the
report's own SQL): 5 documents / 4140.0 for a supplier holding data in
all four sources, whose vouchers carried 3 child lines, so a fan-out
would have shown 7.

**Value basis is GROSS**, per explicit decision. Only Purchase Voucher
has a VAT concept; the other three record what was paid, so their
per-unit `purchase_price` is multiplied by `qty` (Purchase Entry is the
exception — one phone per document, so its price is already the total).
Purchase Voucher contributes its **parent** `total_purchase_value`,
never a re-sum of its child lines, so the figure stays the same one its
print format reconciles against the supplier invoice.

Other decisions worth not relitigating: rows in Purchase Report come
from three doctypes, so `name` is a **Dynamic Link** against a new
`Source` column rather than a fixed Link to Purchase Entry; a new **Qty**
column exists because a row no longer means one handset beside a
per-unit price; a voucher phone line shows `"3 of 5 captured"` while the
IMEI filter searches the underlying `captured_imeis` blob; the brand
filter is `LIKE` because voucher brands are free text carrying the model.
Purchase Report's six filters had been **dead** — declared nowhere in the
JSON, so no UI existed to set them — and are now declared.

Two pre-existing bugs fixed along the way, both with their own commits:
the `LEFT JOIN`/`WHERE` defect that silently deleted zero-purchase
suppliers (now **Hard Rule 17**), and Purchase Report's dead filters.

Verified as the real non-superuser accounts throughout, never
`Administrator`: 27 + 21 + 16 automated checks, legacy Purchase Entry
rows confirmed field-by-field identical to a pre-change baseline, drafts
and cancelled excluded across all four sources, and a browser pass on
both reports as both roles. A real user-entered voucher (`PV-00001`,
supplier Midhu) was in both reports throughout and its 6,750.000 total
was reconciled by hand against the document itself — old-format and
new-format data side by side, so a fix that dropped legacy rows would
have failed visibly.

**Unified `Purchase Voucher` intake — built 2026-07-28, browser pass
outstanding** (11 commits, `cf2fe7f`..`e1cc0d7`; plan at
`~/.claude/plans/new-feature-scoping-for-immutable-puppy.md`). Direct
client request from a real meeting: intake split across three doctypes
felt fragmented, and one supplier delivery of "10 phones + 10 earphones
+ 15 cases" needed three documents under three numbering series. Now one
submittable parent (`Purchase Voucher`, naming `PV-.#####` → `PV-00001`,
plain sequential per the client's "p1, p2, p3") with two typed child
tables, `Purchase Voucher Phone Line` and `Purchase Voucher Accessory
Line` — Shop Sale's proven shape, not a new design.

**IMEI capture is optional per unit.** Whatever staff scan becomes a real
tracked `Phone`; the remainder becomes untracked `Phone Batch` quantity,
identified at the till by the existing `pos_scan()` → Shop Sale flow.
Two hard blocks guard that split, both compliance-driven rather than
arbitrary: a **Used** line must be fully captured (Phone Batch has no
`phone_type` and is New-only by construction — `get_sale_scheme()` and
`validate_no_pms_standard_mix()` both treat a batch row as New/Standard
*without a lookup*, so an untracked Used unit would sell under standard
VAT instead of the margin scheme); and any remainder **requires the box
UPC**, because untracked units are reachable only through `pos_scan()`'s
UPC lookup — a batch with no scannable UPC is permanently unsellable
stock, so the rule is what makes the remainder usable at all.

**Hard rule 17 territory — VAT treatment is PER LINE here, and that is
deliberate.** The first scoping had a document-level `vat_treatment`
mirroring `Shop Sale`, flagged as an assumption. Real accountant feedback
settled it the other way: one distributor invoice can legitimately price
some lines VAT-inclusive and others exclusive. So `vat_treatment` lives
on **both child tables and nowhere else**. **Do not "fix" this into
consistency with Shop Sale's document-level field** — the asymmetry is
the requirement. There is also deliberately **no** `default_vat_treatment`
convenience field on the parent: an earlier draft had one and it was
dropped, because a stored parent value is something a later report or a
careless `frappe.db.get_value` could mistake for authoritative. The
keystroke-saving pre-fill is pure JS session state in
`public/js/purchase_voucher.js`, keyed to the current document so it
cannot leak across vouchers. A proposal to add such a field back is a
re-litigation, not a new idea.

Per-line `net_amount`/`vat_amount`/`amount` are computed by the **shared**
`calculate_standard_vat()` in `mobile_shop/utils/vat.py` — called, never
modified (its second return value is named `net_profit` for its sale-side
caller; on the purchase side the same number is the net cost). A Used
line takes an explicit branch that never calls it at all. Two rounding
decisions, both found by testing and both worth keeping: round **per
line** rather than once at the end (an Inclusive line divides by 11 and
the float tails made displayed lines not add up to the displayed total),
and round only **two** of net/VAT/gross and derive the third, so
`net + VAT == gross` holds exactly — rounding all three independently
produced a visible 3-fils discrepancy across 35 lines. Which figure is
authoritative follows the treatment: Exclusive means the entered price is
net, Inclusive means it already includes VAT.

`before_cancel`/`on_cancel` shipped **in the same feature**, per Hard
Rule 16 — not added afterwards the way all three older intake doctypes
needed. `before_cancel` reuses `PurchaseEntry.get_cancel_block_reason`
verbatim so both intake paths refuse on identical conditions; both stock
decrements sit inside a single `SELECT … FOR UPDATE` lock-check-decrement
rather than split across the two hooks. Note `phones_created` is a
`Small Text` (one line can create many phones), so **Frappe's
link-integrity check cannot see those references at all** — same blind
spot as `Sales Entry.imei`; `before_cancel` is the only guard.

`cancel` granted to `Mobile Shop Admin` and `System Manager` only, never
Staff — matching the 2026-07-25 decision that intake reversals are
stock-provenance corrections.

**No report changes were needed**, and that was verified rather than
assumed: a Phone created here is byte-identical in shape to one from
`Purchase Entry`, and `pos_scan()` resolves both a new tracked IMEI and a
new batch UPC with zero code changes. **But** `Purchase Report` and
`Supplier Report` read `Purchase Entry` exclusively, so they will go
empty for new data — a real gap, deliberately left to its own phase.

Two new print formats (`Self-Billed Purchase Voucher Invoice`,
`Purchase Voucher Summary`). The self-billed one renders **Used lines
only** — that is what makes allowing mixed vouchers correct rather than
merely asserted — says so explicitly when the voucher also holds
distributor stock, and renders a short explanation instead of a blank
signature line when there are no Used lines at all.

Three deliberate consequences to remember: `Phone.model` is **blank** for
everything created this way (the client asked for brand and model folded
into one free-text field), so Inventory/Sales Report's Model column is
empty for new stock and Sales Report's Model filter will not match it;
`Phone Batch.model` was relaxed to optional for the same reason;
and `Phone.purchase_price` stores the price **exactly as entered**, not
grossed up on an Exclusive line — open for the accountant, but it only
feeds the PMS margin for Used phones, which carry no `vat_treatment` at
all.

Verified against the real non-superuser accounts throughout — Staff
`clashams4@gmail.com`, and a **new** `Mobile Shop Admin`-only account
(see Hard Rule 14's update below). Every rejection path was exercised,
not just the happy ones; the two that matter most are that a blocked
batch cancel leaves `untracked_qty` **and** `docstatus` unchanged
(proving the throw rolls the decrement back), and the compound case where
cancelling a batch-originated Shop Sale returns the Phone to In Stock but
does **not** restore `untracked_qty`, so the voucher cancel stays
correctly blocked and the two reversal mechanisms never double-count.
The client's own case — 10 phones + 10 earphones + 15 cases on one
voucher — was submitted end to end through the ordinary permission path
as the real Staff account.

**Three pre-existing bugs found while doing this, all unrelated to the
feature**, plus one worth knowing:

- **`Mobile Shop Admin` had zero permission on `Supplier`** — see Hard
  Rule 14's update. Fixed in `cf2fe7f`.
- **A dangling `User Permission`** restricting Staff to `Supplier = "Ali"`,
  created 2026-07-07 and orphaned by the 2026-07-25 wipe, which deleted
  every Supplier. Staff could therefore read or print **no** supplier-linked
  document at all. Proved by construction (with a supplier named "Ali",
  Staff read/print `True`; with any other, `False`). Swept the whole site
  — it was the only `User Permission` row — and deleted it. **The wipe's
  own "explicitly untouched" audit never covered `User Permission`;
  include it next time.**
- **`bench export-fixtures` silently loses rows when two `fixtures`
  entries name the same doctype.** Both write `fixtures/<doctype>.json`
  and the second overwrites the first — a two-entry version dropped all
  11 `Item` `Custom DocPerm` rows. Use ONE entry with an `in` filter:
  `[["parent", "in", ["Item", "Supplier"]]]`.
- **The homepage launcher's tile-rendering JS exists only in the site
  database.** It is a `Custom HTML Block` record, and `hooks.py` tracks
  only Custom Field / Custom DocPerm / Print Format — so that script is
  untracked by git and would be lost on a fresh install. Same drift class
  as the workspace problem fixed in `d8c4a4b`. Not fixed; flagged.

**Full test-data wipe, 2026-07-25** (no code changes — a database
operation, done on explicit instruction ahead of a full end-to-end test
scenario run from the shared test document). Run through a script rather
than the desk UI specifically *because* of the intake-cancel work
committed the same day: bulk-cancelling old test data through the UI
would now correctly hit the new `before_cancel` blocks wherever a phone
or stock item had already been touched by a sale. The fix working as
designed is what made the UI route unusable for a bulk teardown, which
is worth anticipating next time a mass cleanup is needed.

`bench --site mobileshop.local backup` first, and confirmed genuinely
restorable before deleting anything — not just trusting the "successfully
completed" line: `gzip -t` passed and the archive was checked for the
real `CREATE TABLE`/`INSERT` content
(`20260725_094524-mobileshop_local-database.sql.gz`, 1.0M).

Deleted, in dependency order (transactions before the masters they
reference), all confirmed 0 afterwards by raw SQL `COUNT(*)` in a fresh
connection rather than the wipe script's own reporting: Shop Sale (14),
Sales Entry (2), Purchase Entry (2), Item Purchase (3), Phone Batch
Purchase (0), Phone Batch (0), Phone (2), Item Barcode (1), Item (1),
Customer (3), Supplier (1). Child rows went with their parents — `Phone
Sale Item` 12→0, `Item Sale Line` 6→0, no orphans. Submitted documents
had to be dropped to `docstatus = 2` via SQL first, since Frappe won't
delete a submitted doc; `force=True` on `delete_doc` skipped the
linked-document check, which is correct here only because the whole
reference graph was being removed together.

Two things worth remembering:

- **One record was deleted that wasn't on the list**: a single `Item
  Price` row (Standard Buying, 150.0). ERPNext auto-creates it from
  `Item.standard_rate`, and it would have blocked the `Item` delete and
  otherwise been left orphaned. Any future Item teardown needs the same
  step. Disclosed at the time, recoverable from the backup.
- **The Item `8906129030572` is gone**, superseding the explicit
  leave-it-alone decision recorded in the batch-phones section above —
  see the note appended there.

Explicitly untouched, verified by before/after counts: User 4, Role 51,
Custom Field 3, Custom DocPerm 19, Print Format 27, Workspace 24, Page
17, Report 197, Number Card 29 — all identical.

`Walk-in Customer` re-seeded afterwards (the POS needs it for
accessory-only sales), reproducing the deleted record's field values
captured beforehand rather than guessed — every other field on it was
blank. Verified at both layers: `is_walk_in` reads 1 through the ORM and
1 as the raw column value, and it is the only Customer in the database.

Cache cleared and bench restarted. Smoke-checked that the app behaves
correctly against an empty DB rather than assuming: `pos_scan()` on the
just-deleted barcode returns a clean `{"type": "not_found"}` instead of
erroring.

**Cancel support on all three intake doctypes — built and API-verified
2026-07-25, browser pass still outstanding** (3 commits: 6519b96,
cedc6a7, eb7c59f). Started from a user report of "I can't cancel a
submitted Purchase Entry" and a request to check whether it was the same
gap already found on Shop Sale. It was — and the diagnosis pass found it
on all three intake doctypes at once, making the Shop Sale finding a
*class* of bug rather than a one-off (see the expanded Hard Rule 14 and
the new Hard Rule 16). `Purchase Entry`, `Item Purchase`, and `Phone
Batch Purchase` all had `submit: 1` but no `cancel` key on any
permission row, so Frappe defaulted it to 0 and nobody could cancel any
of them; verified against real non-superuser accounts, not
`Administrator`, which reported `cancel=True` for all three and is
exactly what hid it.

The important half was not the permission. **None of the three had any
`on_cancel`/`before_cancel` reversal logic at all**, so granting cancel
alone would have converted a blocked bug into reachable silent data
corruption. Each got reversal or refusal:

- **Purchase Entry** — `on_cancel` deletes the Phone its `on_submit`
  created (clearing `phone_created` *first*, since that Link field would
  otherwise make Frappe's own link-integrity check refuse the delete).
  `before_cancel` hard-blocks when the unit carries history: a submitted
  `Sales Entry` or `Phone Sale Item` on that IMEI, or any `Phone.status`
  other than `In Stock`. Checked all three ways rather than any one:
  status alone is resettable by hand via the unsupported "return" path,
  and Reserved/Returned means history even with no sale behind it.
  Worth remembering — **nothing at the framework level protects this**:
  `Purchase Entry.phone_created` is the ONLY Link field to `Phone` in
  the entire schema, and both `Sales Entry.imei` and `Phone Sale
  Item.imei` are plain `Data` fields, so link integrity would happily
  let a sold Phone be deleted. `before_cancel` is the only guard.
  Also corrected a premise from the original report: Profit Report reads
  `psi.margin`/`se.margin`, snapshotted onto the sale line at submit
  time, NOT recomputed from `Phone.purchase_price` — so cancelling a
  purchase would never retroactively move figures already in the report.
  What it destroys is provenance and IMEI History Report (Phone-centric,
  so a deleted Phone drops out entirely). Same conclusion, different
  reason.
- **Item Purchase** — `on_cancel` decrements `Item.current_stock`,
  refusing if that would go negative. Accessory stock is fungible with
  no per-unit identity, so "were *these* units sold?" is unanswerable;
  `current_stock >= qty` is the only meaningful check, and what it
  guarantees is that the shop still physically holds enough to give back.
- **Phone Batch Purchase** — same shape against `Phone
  Batch.untracked_qty`. Sound for a subtler reason worth recording:
  batch units are anonymous until they sell, but `untracked_qty` only
  ever moves down through genuine sales — once a unit sells its IMEI is
  captured and the decrement is permanent, since cancelling a
  batch-originated *sale* deliberately does not return it to the batch.
  So what remains is exactly what can be handed back. Three things
  deliberately NOT reversed, documented on the method:
  `last_purchase_price`/`last_supplier`/`last_purchase_date` (overwritten
  every restock, prior values stored nowhere, feed no VAT or profit
  math), the `Phone Batch` master record even at zero (shared, may be
  referenced by real sale rows, and empty is a normal out-of-stock
  state), and any `Phone` already created out of the batch.

Design decision worth keeping: on both stock doctypes the guard lives
*inside* `on_cancel`'s atomic `SELECT ... FOR UPDATE` lock-check-decrement
rather than as a check in `before_cancel` — splitting them across the two
hooks leaves a window for a concurrent sale to drain stock in between,
the exact race `consume_item_stock`'s `FOR UPDATE` exists to close.
Mirrors `consume_item_stock`/`create_phone_from_batch`, not
`process_purchase`'s bare UPDATE. Confirmed empirically rather than
assumed: on a blocked attempt both the stock figure and `docstatus`
were unchanged, so the throw really does roll the decrement back.

`cancel` granted to `Mobile Shop Admin` and `System Manager` only, never
`Mobile Shop Staff` — explicit user decision: these are stock-provenance
corrections, unlike the POS's 15-minute staff void window on Shop Sale.

Verified as the real non-superuser accounts (Hard Rule 14) across every
rejection path, not just the happy ones: both real sold phones blocked
with correct messages, Reserved-status blocked, clean cancels deleting
the Phone / decrementing exactly, dangling-`phone_created` case cancelling
cleanly, both boundaries on each stock doctype (`stock == qty` succeeds,
`stock == qty-1` blocks), per-purchase reversal against a shared restocked
batch counter, and Staff blocked outright on all three. The compound case
is the one worth re-running if this code is ever touched: cancelling a
batch-originated Shop Sale returns the Phone to `In Stock` but does NOT
restore `untracked_qty`, so the purchase cancel stays correctly blocked —
confirming the two reversal mechanisms don't double-count. Stable across
two migrates each; no `Custom DocPerm` rows on any of the three, so Hard
Rule 15 isn't in play.

All test data cleaned up, confirmed zero residue, real data re-verified
intact (3 Purchase Entries at docstatus 1 with phones attached, 3 Item
Purchases, item stock 29). One disclosure: exercising the Item Purchase
path against real data meant cancelling `IP-2026-07-00009` and restoring
it via raw SQL — values are exactly as before, but its `modified`
timestamp now reads 2026-07-25 and `modified_by` is the admin account.
Phone Batch Purchase was tested on synthetic data only. (Both the data
described here and that lingering timestamp were removed hours later by
the full wipe above — the paragraph records what was verified at the
time, not current state.)

**Full interactive end-to-end verification pass, 2026-07-22** (no code
changes — a dedicated live-browser QA pass across the whole app, both
roles). Homepage tiles (14 Admin / 12 Staff, one nav check per tile type),
all 8 POS cart scenarios (New/Used/accessory sales, every mixing-block
combination, already-sold IMEI, insufficient stock, walk-in+phone block,
a completed mixed sale with both print formats verified line-by-line),
all 8 reports loading with real data as Admin and the two Admin-only ones
hard-blocking for Staff with a genuine `PermissionError` (not a blank
page), and a Staff-side Item Purchase submit confirming `current_stock`
increments and `purchase_price` stays editable. One real, pre-existing,
previously-undocumented finding: the POS's Print Receipt/Print Invoice
buttons call `frappe.utils.print()` with `trigger_print=1`, which fires the
native OS print dialog and blocks the tab — expected for a real print
button, but worth knowing before ever automating this flow again (verify
print content instead via the same `/printview` URL with
`trigger_print=0`). No functional bugs found. Test data (2 purchased
phones, 2 completed sales, one accessory stock bump) left in place as real
data per explicit instruction, not cleaned up — subsequently removed by
the full wipe of 2026-07-25.

**Customer phone search/dedup + three POS polish pieces, 2026-07-22/23**
(commits 35be8e9, 7c389e0, 5dd6acb, 654a01e). `Customer.search_fields =
"phone"` (every existing Customer Link field now autocompletes on phone,
free); a POS "Search by Phone" button mirroring the existing "Search Phone
by Brand/Model" pattern (`search_customers_by_phone()`, plain
`frappe.get_all` — Customer has no permlevel fields to bypass); a soft,
non-blocking duplicate-phone warning in `Customer.validate()` (deliberately
on the document, not just the POS, so the Link field's own inline
"+ Create a new Customer" quick-entry catches it too) — editing this
button into `mobile_shop_pos.js` is what surfaced Hard Rule 13 (a stale
Page-script Redis cache that survived migrate/restart/hard-refresh). A
browse-accessories panel filling the POS's previously-empty left-column
space, ranked by sale-frequency-then-recency (`browse_items()`, folds
"recent/frequent" into the default sort with no separate UI needed) and
calling `add_accessory_line()` directly — the identical path barcode
scanning uses, never a second implementation. A brief green flash + bundled
`frappe.utils.play_sound('click')` on a successful scan only (not on
every `scan_ok()` message, so "Cart cleared." stays silent). A "Void
Sale" button in the POS success banner, backed by a new
`ShopSale.on_cancel()` (closing a real pre-existing gap — cancelling used
to do nothing at all, leaving phones stuck "Sold" and stock permanently
short) and `ShopSale.before_cancel()` as the actual enforcement (Admin/
System Manager unrestricted; Staff only their own sale, only within
`STAFF_VOID_WINDOW_MINUTES` of submission) — building this last piece
surfaced Hard Rule 14 (verifying "Admin" against the `Administrator`
superuser masked a missing `cancel` permission grant on the real roles)
and a second real bug caught only by testing the actual rejection paths:
the window check first used `self.modified`, which Frappe itself
overwrites to *now* before `before_cancel` ever runs, so a months-old sale
slipped through unblocked until fixed to `self._original_modified`. All
four verified against real accounts, including a genuine non-superuser
Admin — not just `Administrator` — per Hard Rule 14.

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

**Small unfinished items**: the POS-related items in "Next task" above
(tablet camera-scanner path, one real print once the thermal printer
arrives — the Staff-role no-leak pass this used to also list is done as
of 2026-07-22) — otherwise the POS build is considered done;
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

**Batch-received New phones (untracked IMEI stock) — built and fully
verified 2026-07-23, including a real interactive browser pass** (plan at
`~/.claude/plans/elegant-sniffing-perlis.md`,
3 commits: d255caf, 554f330, de40d58). New-phone-only path alongside the
existing per-unit Purchase Entry flow (untouched — still used for Used
phones and for New phones bought singly): scan a box's model-level UPC,
enter a quantity, done — no per-unit IMEI scanning until the phone actually
sells, when staff capture the real unit's IMEI in the POS cart itself.
Mirrors Phase 1's `Item`/`Item Purchase` shape rather than reusing `Item`
directly (considered and rejected — `Item` has no phone-shaped attributes
and its cost fields aren't permlevel-1 like `Phone.purchase_price` is):
new `Phone Batch` (master, not submittable, `autoname: "field:upc"` so the
UPC value doubles as the doctype name, `untracked_qty` mirroring
`Item.current_stock`, `last_purchase_price` permlevel 1) and `Phone Batch
Purchase` (submittable intake transaction mirroring `Item Purchase`'s
shape exactly, including its convention of granting no one `cancel`).
Name-collision check run against `apps/frappe`/`apps/erpnext` before
building — no conflict; ERPNext's own core `Batch` doctype is a different,
unused concept (stock-ledger lot tracking).

`Phone Batch.validate()` rejects a UPC already registered as an accessory's
`Item Barcode` — a real guard against the box-photo collision risk this
feature was built to handle, though an explicitly accepted limitation:
it only catches that direction, not an `Item Barcode` added *after* a
`Phone Batch` already exists for the same value, since `pos_scan()` checks
`Item Barcode` first. `pos_scan()` gained a third lookup branch (UPC ->
Phone Batch) alongside its existing Item-Barcode/Phone-IMEI checks. New
`Phone Sale Item.phone_batch` field (blank on every normal line) is how
`ShopSale.process_phone_line()` tells a batch-originated cart line apart
from a normal one; for a batch line, `ShopSale.create_phone_from_batch()`
locks-checks-decrements `untracked_qty` (same `FOR UPDATE` pattern as the
existing `consume_item_stock()`) then creates the real `Phone` record via
`phone.insert()` — reusing `Phone.validate()`'s IMEI format/Luhn/uniqueness
checks unchanged, the same call `PurchaseEntry.create_phone_record()`
already makes. `get_sale_scheme()`/`validate_no_pms_standard_mix()` both
had to learn to treat a `phone_batch`-tagged row as New/Standard without a
DB lookup, since no Phone record exists for it yet at `validate()` time —
a real gap caught before it could ship, not found by testing.

No changes needed to any of the 8 existing reports: once sold, a
batch-originated unit is a completely normal `Phone` + `Phone Sale Item`
row, indistinguishable from a Purchase-Entry-sourced one. Confirmed
directly — a real batch sale showed up correctly in IMEI History Report
with zero code changes.

Live-DB investigation before building turned up a real finding worth
remembering: the UPC from the actual box photo driving this feature
(8906129030572) already exists in this database as an `Item` named
"Redmi note 10" — but it's not idle stray test data as originally assumed;
it has 3 real submitted `Item Purchase` documents and 6 real `Shop Sale`
documents against it. Decision: leave it alone entirely (confirmed with
the user, not a unilateral call) — it has no `Item Barcode` row so it
doesn't functionally collide with `pos_scan()` today, and deleting/renaming
it would have broken those real Link references. `Phone Batch` will still
be named `"8906129030572"` in its own doctype namespace, a human-readable
overlap accepted as a known quirk, not a hard conflict.

**Superseded 2026-07-25 — that Item no longer exists.** The
leave-it-alone decision above held only as long as those Link references
did. The full test-data wipe (see the dated entry below) deleted the
`Item Purchase` and `Shop Sale` documents that made deleting it risky,
and the Item itself along with them, on explicit instruction naming it as
stray test data to clear. The paragraph above is kept for the reasoning,
not as a description of current state: there is now no `Item` named
`8906129030572`, so the human-readable overlap with a future `Phone
Batch` of the same name is gone too. Nothing about the UPC-collision
guard in `Phone Batch.validate()` changes — it checks `Item Barcode`
rows, and that Item's barcode was `8888888888`, never the UPC itself.

Verified end-to-end as the real Mobile Shop Staff account
(`clashams4@gmail.com`, not `Administrator` — Hard Rule 14), including two
transactional edge cases the plan specifically called for, tested via real
HTTP requests rather than bench console (a bench console session doesn't
share a request's commit/rollback wrapper, confirmed empirically — test
data written via console with no explicit `frappe.db.commit()` silently
never persisted across sessions): (1) cancelling a batch-originated sale
restores the Phone to "In Stock" and leaves `untracked_qty` untouched — the
unit stays permanently tracked once its IMEI is captured, never returned to
the anonymous batch count, confirmed by design decision rather than
accident; (2) a duplicate IMEI submitted via a real HTTP POST fails inside
`Phone.validate()`'s existing uniqueness check, which runs *after* the
batch's atomic decrement within the same request — confirmed the whole
transaction rolls back (`untracked_qty` unchanged, no orphaned draft `Shop
Sale` left behind), the single most likely real-world failure mode (a
mistyped IMEI at the till) proven safe rather than assumed safe.

All of the above was exercised through the real Python API layer
(`pos_scan`/`create_pos_sale`/`void_pos_sale`, called directly and via
actual HTTP requests) and the real permission/field-redaction code paths —
not code review, not a dry run.

**Interactive browser pass, 2026-07-23**, logged in as the real
`clashams4@gmail.com` session already open in the browser (confirmed via
`/app/user-profile` before touching anything, not assumed): created and
submitted a real `Phone Batch Purchase` through the actual desk form
(UPC/brand/model/storage/color/qty/supplier/price, BHD 3-decimal precision
confirmed in the rendered field), confirmed the `Phone Batch` back-reference
field populated. In the POS: scanned the UPC, got the exact designed cart
line (PHONE badge, "from batch · New", IMEI input with placeholder, camera
button, the orange no-lookup-exists warning); picked a named customer via
the phone-search autocomplete; entered a unit IMEI and price; completed the
sale — real green success banner, Print Receipt/Print Invoice/Void Sale
buttons rendered. Verified the receipt via `/printview?...&trigger_print=0`
(never clicked the actual print buttons — they call `frappe.utils.print()`
with `trigger_print=1`, which blocks the tab, a known pitfall from the
original POS build) — correct brand/model/IMEI/VAT math, no "None" bugs.
Clicked the real Void Sale button (a `frappe.confirm` modal, not a native
dialog, safe to click) and confirmed server-side the Phone returned to "In
Stock" with `untracked_qty` untouched, matching the API-level test exactly.
Also drove three client-side guards live: scanning the same UPC past
remaining stock produced the exact "Only N unit(s) ... are in untracked
stock" message; completing with a walk-in customer produced the phone-needs-
named-customer block; completing with prices set but IMEIs blank produced
"Enter or scan the unit IMEI for every phone" (price-before-IMEI check order
confirmed by triggering the price message first, then clearing it). Clicked
the per-line camera scan button itself - correctly opened the scanner and
fell back to a clean "Camera Unavailable" message (no real camera in this
environment), no JS crash; console showed only the pre-existing, expected
`imei_scanner.js` diagnostic logging for that fallback path, nothing from
the new batch code. All test data (the Phone Batch Purchase, Phone Batch,
and every test Phone record) cleaned up afterward, confirmed gone.

**Homepage tiles for Item Purchase/Phone Batch Purchase/Item/Phone Batch —
added and verified 2026-07-23** (commit ba57cee). The four doctypes built
for the batch-phones feature and Phase 1's accessories had no launcher
tile — same class of gap as the earlier Workspace shortcut drift. Added to
`TILE_SECTIONS` in `mobile_shop/utils/home_tiles.py`: Item Purchase and
Phone Batch Purchase under Daily Tasks, Item and Phone Batch under
Records. No new visibility logic needed — `can_see_shortcut()`'s existing
`frappe.has_permission()` check already gates all four correctly once the
underlying doctype permissions are right.

**Found and fixed a real, previously-undocumented permission bug while
verifying the new Item tile against a real non-superuser admin account
(Hard Rule 14 discipline, not `Administrator`)**: Frappe treats *any*
`Custom DocPerm` row on a doctype as a **complete override** of that
doctype's permissions, not an addition to the standard ones. The single
`Mobile Shop Staff` read-only `Custom DocPerm` row added for `Item` back
in Phase 1 had silently wiped out all of core ERPNext's standard `Item`
permissions (Item Manager, Stock Manager, Stock User, Sales User, Purchase
User, Maintenance User, Accounts User, Manufacturing User) for every other
role, site-wide — invisible until now because every prior "Admin" check in
this project's history used either `Administrator` (bypasses all
permission checks) or `Mobile Shop Staff` (the one role that still
worked). Confirmed empirically before touching anything: live DB had
exactly one `Custom DocPerm` row on Item; `get_valid_perms('Item',
user=<real admin>)` returned nothing at permlevel 0 despite that account
holding "Item Manager" and several other roles core ERPNext's own
`item.json` grants read to.

**Fix**: re-declared all 8 original roles as `Custom DocPerm` rows (exact
field values copied from `erpnext/stock/doctype/item/item.json`, not
guessed) plus explicit `Mobile Shop Admin`/`System Manager` rows with full
CRUD, matching the admin-tier pattern every other mobile_shop-owned
doctype already uses. Widened the Item `Custom DocPerm` fixture filter in
`hooks.py` from role-scoped (`role = Mobile Shop Staff`) to parent-scoped
(`parent = Item`) so all 11 rows stay fixture-tracked going forward, not
just the one that caused this. Regenerated `fixtures/custom_docperm.json`
via `bench export-fixtures`, confirmed stable across two migrates.

**This is now a new hard rule worth remembering — see Hard Rule 15**:
adding a `Custom DocPerm` row for one role on a core doctype silently
disables every other role's standard access to that doctype. Always check
whether the target doctype already has *any* `Custom DocPerm` rows before
adding one, and if it doesn't yet, be prepared to re-declare the full
original role set, not just the one role being added.

**Swept for the same failure mode elsewhere**, per explicit instruction:
`Supplier` is the only other core doctype this app has `Custom DocPerm`
rows for. Checked and confirmed healthy — its existing 8 rows (Accounts
Manager, Accounts User, Mobile Shop Staff, Purchase Manager, Purchase
Master Manager, Purchase User, Stock Manager, Stock User) already cover
the full role set a real admin account needs; `has_permission` verified
`True` for both read and write against the real admin account. (Separately
noted but not touched, since it's not the same problem: Supplier's rows
aren't fixture-tracked in `hooks.py` at all, unlike Item's — a
pre-existing, already-documented distinction from the original Phase 1
work, not a new finding.)

**Verified via the real API for both real accounts** (`clashams4@gmail.com`
/ Mobile Shop Staff, `abhijithms.9526@gmail.com` / System Manager — not
`Administrator`): correct tile lists for both roles (`get_homepage_tiles()`
called directly as each user), Item/Phone Batch read-only for Staff vs full
CRUD for Admin (`has_permission` checked explicitly for read/write/create),
stable across two migrates. **Browser click-through**: Staff verified via
the already-logged-in session — all four tiles navigate correctly, correct
icons/colors, `+ Add` button present on Item Purchase/Phone Batch Purchase
and correctly absent on Item/Phone Batch. Admin verified via `Administrator`
for the UI-rendering side only (permission correctness already proven
separately at the API layer against the real account, per the project's own
carve-out for non-permission UI checks) — all four tiles present, `+ Add`
button correctly present on Item/Phone Batch this time. No new console
errors from either pass (only the pre-existing, expected camera-unavailable
diagnostic logging from the earlier batch-phones session).

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

**Added 2026-07-28, from the Purchase Voucher work** — four more, none
urgent, none to be decided unilaterally:

- ~~`Purchase Report` / `Supplier Report` do not see Purchase Vouchers.~~
  **Done 2026-07-29.** ~~Should accessory purchases be reported at all?~~
  **Answered — `Accessory Purchase Report` built the same day.**
  ~~The purchase-side reports are stricter than their own doctypes.~~
  **Also answered 2026-07-29:** the three purchase-side reports were
  relaxed to permlevel 0 and now show money to Staff (commit `2037845`).
  Sales / Profit / VAT Report unchanged. See Hard Rule 3's table.
- **What `Phone.purchase_price` stores on a VAT-exclusive line** — as
  entered (current behaviour) or grossed up. The accountant's call. It
  only ever matters for New phones, since Used lines carry no
  `vat_treatment` and it is New-phone `purchase_price` that feeds no VAT
  or profit maths.
- **Whether a voucher mixing Used and standard-VAT lines should hard-block
  rather than warn.** Currently a soft `msgprint`. Becomes a one-line
  change to `frappe.throw` if the client confirms a Supplier is always
  either a private individual or a registered distributor, never both.
- **Whether `Phone.model` staying blank is acceptable** for stock created
  through Purchase Voucher, or whether Sales/Inventory Report's Model
  column and filter should fall back to searching `brand`.

**Added 2026-09-02, from the accountant meeting and the new accounting/POS
feature set** (full detail in PROJECT_PLAN.md's "Accounting & POS Feature
Set — Phase Plan" section):

- **Frontend vs. backend sequencing is unresolved.** Phases 3–6 of the new
  accounting work (money in/out, the three registers) build Desk-side forms
  that the planned custom React frontend would then need to re-implement.
  Whether the accounting layer or the frontend rebuild goes first has not
  been decided — do not assume Desk is the permanent target, and do not
  pick an order unilaterally.
- **The accounting layer's architecture is SETTLED, not open** — listed
  here only so it isn't mistaken for a pending decision: fully bespoke,
  GL-ready `Bank` master + payment child table, deliberately NOT ERPNext's
  native `Bank Account`/`Mode of Payment`/`Payment Entry`. Full reasoning in
  PROJECT_PLAN.md. Before ever proposing "just use ERPNext's native Payment
  Entry" here, read that reasoning — it was already weighed and rejected
  (Company/CoA drag-in, the Hard Rule 15 Custom DocPerm trap, the
  custom-React-frontend payoff argument). See Hard Rule 20 for the field
  contract that keeps this decision cheap to migrate later.

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

   **Update 2026-07-29 — `Report` behaves the same way, and the mechanism is
   now understood.** Adding filters to `purchase_report.json` and running
   `bench migrate` left the live record at `filters: []`. The cause is not
   literally "insert-only": `frappe.modules.import_file` **skips any file whose
   JSON `modified` timestamp is not newer than the DB record's**. Editing the
   file without touching `modified` therefore changes nothing, silently, with a
   clean migrate log. Two ways out:
   - **bump `modified` in the JSON** (what was done — a real edit deserves a
     real timestamp, and it makes fresh installs correct too); or
   - force it: `from frappe.modules.import_file import import_file_by_path;
     import_file_by_path(path, force=True)`.

   This very likely explains some of the historical Workspace/Number Card
   "insert-only" pain too — same code path, same skip.

   **Also: `developer_mode` is OFF on this site.** Standard Reports cannot be
   saved through the ORM at all (`Standard reports can only be created in
   developer mode`), so `frappe.get_doc("Report", ...).save()` is not an option
   — use `import_file_by_path`, or `frappe.db.set_value` for plain columns.
   Note `Report.filters` is a **child table** (`Report Filter`), not a column,
   so `db.set_value` does not work on it.

2. **Script Report folder/file names are derived automatically from the
   Report's `name` field** (scrubbed: lowercased, spaces→underscores). A
   report named "VAT Report" MUST live in folder `vat_report/` with file
   `vat_report.py` — not a shorter or "cleaner" name. Wrong folder name causes
   `ModuleNotFoundError` at runtime even if the JSON name field is correct.

3. **`"permlevel": 1` on a Script Report column enforces NOTHING by itself.**
   Script Reports run raw SQL directly, bypassing normal document-level field
   permissions entirely. Any sensitive field must be omitted ENTIRELY from
   both the `columns` list and the SQL SELECT clause when the user isn't
   admin — never fetched, never a NULL placeholder. Use a shared
   `has_admin_role()` pattern:
   `bool(set(frappe.get_roles(frappe.session.user)) & {"Mobile Shop Admin", "System Manager"})`.
   For fully Admin-only reports (e.g. Profit Report, VAT Report): the JSON
   `roles` array must list ONLY Admin/System Manager (no Staff), AND
   `execute()` must call `has_admin_role()` as its literal first line and
   `frappe.throw(_("Not permitted"), frappe.PermissionError)` if false —
   belt-and-suspenders, not either/or.

   **Which fields are actually sensitive (settled 2026-07-29 — check the
   permlevel, do not go by the field's name):**

   | Field | permlevel | In reports |
   |---|---|---|
   | `Phone.purchase_price` | **1** | Admin only |
   | `Sales Entry` / `Phone Sale Item` — `margin`, `vat_amount`, `net_profit` | **1** | Admin only |
   | `Item Sale Line.vat_amount` | **1** | Admin only |
   | `Purchase Entry` / `Item Purchase` / `Phone Batch Purchase` — `purchase_price` | **0** | **Staff too** |
   | `Purchase Voucher` line + parent money fields | **0** | **Staff too** |

   The trap is that `purchase_price` appears in both halves of that table.
   On `Phone` it is permlevel 1 and feeds the PMS margin; on the intake
   doctypes it is permlevel 0 and Staff type it themselves. Sales Report
   reads it from `tabPhone` (aliases `p` / `p2`) and so stays Admin-only;
   Purchase Report reads it from the intake doctypes and does not.

   The three purchase-side reports were relaxed to match their sources on
   2026-07-29 — they had been stricter than the doctypes for months, which
   protected nothing and made report and form disagree. **Before gating or
   ungating any report column, look up the source field's permlevel.**

   One knock-on, accepted: Staff can see purchase cost and selling price in
   different reports, so per-unit margin is derivable even though `margin`
   itself stays permlevel 1. That was already true — Staff enter purchase
   prices on the intake forms and see selling prices at the POS.

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

    **Update 2026-07-25 — this was never a one-off.** A user report of
    "I can't cancel a submitted Purchase Entry" prompted checking whether
    the same gap existed elsewhere. It did, on every other submittable
    doctype in the app: `Purchase Entry`, `Item Purchase`, and `Phone
    Batch Purchase` ALL had `submit: 1` and no `cancel` key at all. Shop
    Sale was simply the first one anybody happened to try to cancel. Four
    instances of the identical mistake, all written at different times,
    all invisible for the same reason. The generalisable lesson: **when a
    permission-shaped bug is found on one doctype, sweep every sibling
    doctype for it immediately** — the same instinct that Hard Rule 15's
    `Custom DocPerm` sweep came from. A one-line audit catches it:
    `[(d, frappe.get_meta(d).is_submittable, frappe.get_all("DocPerm",
    filters={"parent": d}, fields=["role","submit","cancel"]))
    for d in <app doctypes>]`.

    **Update 2026-07-28 — "not `Administrator`" was never sufficient on
    its own.** This rule says to test against "a real non-superuser
    account carrying the role in question". In practice every later pass
    used `abhijithms.9526@gmail.com`, which holds `System Manager` — so
    `Mobile Shop Admin`, a role named in the permissions array of nearly
    every doctype this app owns, **had never once been exercised**,
    because **no user held it at all**. Discovered while building
    Purchase Voucher, by creating the first such account
    (`msadmin.test@mobileshop.local`, that role and nothing else).
    It immediately found a real bug: **`Mobile Shop Admin` had zero
    permission on `Supplier`** — read, write, create and delete all
    denied. `Supplier` carries `Custom DocPerm` rows, so per Hard Rule 15
    its standard perms are ignored for every role, and its 8 rows named
    `Mobile Shop Staff` plus six core ERPNext roles but neither
    admin-tier role. Since `supplier` is a reqd Link on every intake
    doctype, that blocked those forms **outright** for that role (see
    PROJECT_PLAN Phase 5 bug #2 — a Link to a doctype with no grant kills
    the whole form, not just the field). CLAUDE.md had recorded Supplier
    as swept and healthy; that was true of the *account* used to check it
    (which separately holds `Purchase Manager`/`Stock Manager`) and false
    of the *role*. Fixed in `cf2fe7f`, additive only.
    **The generalisable lesson: a role with permission rows and no
    account holding it is untested, no matter how many non-superuser
    passes have run. Check `frappe.get_all("Has Role", filters={"role":
    r, "parenttype": "User"})` before believing any role has been
    verified**, and keep `msadmin.test@mobileshop.local` around as the
    fixture for `Mobile Shop Admin`. Its DocPerm rows were also diffed
    against `System Manager` across all app doctypes at that point and
    were identical — worth re-running if either is edited.

15. **Adding a single `Custom DocPerm` row for one role on a core doctype
    silently disables every other role's *standard* access to that
    doctype.** Frappe treats the mere existence of any `Custom DocPerm` row
    for a doctype as "this doctype's permissions are now fully custom" —
    the doctype's normal DocPerm rows (whatever core ships, e.g. Item
    Manager/Stock Manager/etc. on `Item`) stop being consulted at all, for
    every role, not just the one the new row names. Discovered when adding
    a homepage tile for `Item`: Phase 1 had added one `Custom DocPerm` row
    granting `Mobile Shop Staff` read-only access, which (invisibly, since
    every later check used either `Administrator` or `Mobile Shop Staff`
    itself) had wiped out core ERPNext's own standard Item permissions for
    every other role site-wide. Confirmed via Frappe's own source
    (`get_valid_perms` in `permissions.py`): once a doctype has any custom
    perms, only custom perms are returned — the standard ones are never
    merged in. **Before adding a `Custom DocPerm` row for a core doctype
    this app doesn't already have one for, check whether it already has
    *any* `Custom DocPerm` rows** (`frappe.get_all("Custom DocPerm",
    filters={"parent": doctype})`). If it doesn't yet, and other roles
    need standard access preserved, re-declare the full original role set
    as `Custom DocPerm` rows (copy exact values from the core app's own
    `<doctype>.json`), not just the one role being added.

16. **Never grant a missing `cancel` permission without first checking
    whether the doctype has `on_cancel` reversal logic — granting it alone
    turns a blocked bug into reachable silent data corruption.** When the
    intake-cancel work (2026-07-25) found `cancel` missing on all three
    intake doctypes, the obvious fix was to add the permission. That would
    have been actively worse than the bug. Every one of them mutates state
    *outside itself* on submit — `Purchase Entry` creates a `Phone`,
    `Item Purchase` increments `Item.current_stock`, `Phone Batch Purchase`
    increments `Phone Batch.untracked_qty` — and not one had any
    `on_cancel`/`before_cancel`. The permission being 0 was the only thing
    preventing orphaned Phone records and permanently overstated stock. So:
    **for any submittable doctype, `on_submit` side effects and `cancel`
    permission are one decision, never two.** Before granting cancel, ask
    what `on_submit` touched and answer for each: reverse it, or hard-block
    the cancel when reversing would destroy something. And prefer a hard
    block over a clever partial reversal when the record carries history —
    a purchase whose phone has been sold should refuse to cancel, not try
    to unwind a sale. Two corollaries learned the same day:
    - Put the guard *inside* the same atomic `SELECT ... FOR UPDATE` block
      as the decrement, not in `before_cancel` with the decrement in
      `on_cancel` — split across the two hooks leaves a window for a
      concurrent sale to drain stock in between. Throwing from `on_cancel`
      aborts the whole cancel and rolls the decrement back (verified: on a
      blocked attempt both the stock figure and `docstatus` were unchanged).
    - Frappe's link-integrity check will refuse to delete a record another
      doc still Links to, so clear the referring field *before* the delete
      — but do not rely on that check as a safety net. It only sees real
      `Link` fields, and this app's sale rows reference phones through
      plain `Data` IMEI fields, which it cannot see at all.

17. **In a report built on a `LEFT JOIN`, a condition about the JOINED side
    belongs in the `ON` clause, never in `WHERE` — putting it in `WHERE`
    silently converts the LEFT JOIN into an INNER JOIN.** Found 2026-07-29 in
    Supplier Report, which had shipped this way since it was written. Its
    date and phone_type filters read
    `LEFT JOIN \`tabPurchase Entry\` pe ... WHERE pe.purchase_date >= %(from_date)s`.
    For a supplier with no matching purchase `pe.purchase_date` is `NULL`,
    `NULL >= '2026-07-01'` is not true, and **the whole supplier vanished from
    the report** instead of showing 0.

    What made it dangerous is that it looked like it worked. Anyone reviewing
    supplier coverage with a date range set was reading "suppliers we bought
    from in this window", not "all suppliers with their totals for this
    window", and the missing rows left no trace to notice.

    The split to apply: **a condition that decides which rows are LISTED goes
    in `WHERE`; a condition that only decides which joined rows COUNT goes in
    `ON`.** In Supplier Report, supplier-level filters (disabled, supplier
    name) are WHERE; purchase-level filters (date, phone_type, docstatus) are
    ON.

    Two corollaries learned with it:
    - After fixing this, **check the filters still actually filter.** It is
      easy to "fix" the join by making the condition a no-op. The test that
      catches it: a narrowing filter must still lower the count, and a window
      containing no data must give every row 0 rather than hiding rows.
    - Use `COUNT(<joined>.name)`, never `COUNT(*)`. A LEFT JOIN with no match
      produces one all-NULL row, which `COUNT(*)` happily reports as 1.

18. **Test-data cleanup must delete only what it created — never sweep a whole
    table.** On 2026-07-29 a teardown script that did
    `for n in frappe.get_all(dt): delete_doc(...)` removed a `Supplier` the
    user had entered by hand minutes earlier. It was recoverable in full from
    Frappe's own `Deleted Document` archive (which stores the complete JSON of
    every deleted doc — remember this, it is the first thing to check), but
    only because it was noticed immediately.

    This was safe for months purely because the database was empty after the
    2026-07-25 wipe. The moment real data exists, table-sweeping teardown is
    destructive. **Write a manifest of created record names and delete from
    that**, and collect side-effect records (Phones created by a submit,
    `Item Price` rows auto-created from `Item.standard_rate`) by tracing them
    from the manifest rather than by listing the table.

19. **`create` without `write` is not a usable permission combination — a
    Frappe form cannot render a blank field the user is not allowed to write.**
    On 2026-07-29, `create: 1` was granted to `Mobile Shop Staff` on `Item`
    (and nothing else, deliberately) to unblock the inline
    "+ Create a new Item" dialog on a Purchase Voucher accessory line. It could
    never have worked. `frappe.perm.get_field_display_status()` consults only
    `write` when deciding between `"Write"` and `"Read"` — **`create` is never
    looked at** — so with read+create every field resolved to `"Read"`, and
    Frappe then *hides outright* any Read field whose value is null. The fields
    that rendered were exactly the ones that already had values (`item_code`,
    `stock_uom`, the checkboxes); every blank mandatory field
    (`item_group`, `item_name`, `description`) was simply absent. Save failed
    with "Missing Values Required: Item Group" and there was **no field on
    screen to fix it** — in the quick-entry dialog *and* in Edit Full Form.

    The trap is that this looks like a missing permission on the *linked*
    doctype, and chasing that produces real-looking progress: granting UOM
    `select` genuinely did fix a separate "No permission for UOM" error and
    made `Default Unit of Measure` prefill. It just wasn't the blocker. Two
    dead ends were burned before the actual mechanism was found — including
    `Stock Settings.item_group`, which does **not** feed quick-entry the way
    `stock_uom` does (`stock_uom` is published into
    `frappe.boot.sysdefaults`; `item_group` is not published at all).

    **Diagnose this class of bug by asking the form directly, not by reading
    DocPerm rows.** The decisive check takes one line in the browser console
    and changes nothing:
    ```js
    frappe.perm.get_field_display_status(
        frappe.meta.get_docfield(dt, fieldname, docname), doc, cur_frm.perm)
    ```
    Compare it against a synthetic perm array with `write: 1` added. If every
    field flips `Read` → `Write`, the missing grant is `write`, full stop.

    **The fix, when Staff must create but must not edit the master generally,
    is an `if_owner` row** — Frappe core's own pattern, used on `Note`,
    `Kanban Board` and `Custom HTML Block`: **two rows at the same
    role+permlevel**, one `if_owner=0` carrying the unrestricted rights and one
    `if_owner=1` carrying the owner-scoped ones. On `Item` that is
    `if_owner=0: read, create` plus `if_owner=1: read, write` — Staff read
    everything, create anything, write only their own. Note that this is
    invisible to a doc-less permission check: `has_permission("Item", "write")`
    with no `doc` returns `False`, correctly, so **always pass a real document
    when verifying an `if_owner` grant**, and verify both directions (own doc
    writable, someone else's not) with a real `.save()`, not just
    `has_permission`.

20. **Every bespoke payment-related row must carry the GL-ready field
    contract, from the first commit that creates it — not retrofitted
    later.** Settled 2026-09-02 alongside the decision to build a bespoke
    accounting layer instead of using ERPNext's native `Bank Account`/`Mode
    of Payment`/`Payment Entry` (see PROJECT_PLAN.md's "Accounting & POS
    Feature Set" section for the full reasoning). What makes that decision
    safe rather than a permanent dead end is **not** the doctype choice —
    it's the field set. A future Phase A (real GL integration) needs to be
    able to generate a GL entry from each historical payment row with no
    re-capture, so every one of them — Shop Sale's payment child table,
    Payment Voucher's, and any later addition — must carry:
    - date
    - direction (in / out)
    - method
    - amount
    - bank link
    - party
    - the parent document it settles

    Field names deliberately borrow ERPNext's own Payment Entry vocabulary
    so the eventual Phase A mapping is near-mechanical rather than a
    redesign: `mode_of_payment`, `paid_amount`, `party`, `party_type`,
    `reference_doctype`, `reference_name`. Do not invent parallel names for
    the same concepts, even if a shorter or more "obvious" name occurs to
    you at the time — the whole point is that these fields already read as
    Payment Entry fields.

    **Accepted trade-off**, so it isn't rediscovered as a surprise later:
    Phase A will still need a one-time backfill job to produce historical GL
    entries from these rows, since they won't have been accruing natively
    from day one.

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

**Custom doctypes** (13, as of 2026-07-28 — this list was long out of date;
verify with `frappe.get_all("DocType", filters={"module": "Mobile Shop"})`
rather than trusting any prose):

- Masters: `Phone`, `Customer`, `Phone Batch`
- Sale side: `Shop Sale` + children `Phone Sale Item`, `Item Sale Line`;
  `Sales Entry` (historical only, superseded by Shop Sale)
- Intake: `Purchase Voucher` + children `Purchase Voucher Phone Line`,
  `Purchase Voucher Accessory Line` — the single entry point since
  2026-07-28; `Purchase Entry`, `Item Purchase`, `Phone Batch Purchase`
  all still fully functional but historical-only for new entry

Plus 9 Report doctypes (IMEI History, Sales, Purchase, Accessory
Purchase, Customer, Supplier, Inventory, Profit, VAT — all built),
3 Number Cards, 2
Workspaces (`Mobile Shop`, `Mobile Shop Home`), 1 Desk Page
(`mobile-shop-pos`), 1 Custom HTML Block (the homepage launcher — **lives
only in the site DB, not fixture-tracked**), and 9 Print Formats.

**Which reports read which purchase sources** (as of 2026-07-29 — check
the SQL, not this table, before relying on it):

| Report | PE | Item Purchase | PBP | Purchase Voucher |
|---|---|---|---|---|
| Purchase Report | ✅ | ❌ *(deliberate)* | ✅ | ✅ phone lines only |
| **Accessory Purchase Report** | ❌ *(deliberate)* | ✅ | ❌ | ✅ accessory lines only |
| Supplier Report | ✅ | ✅ | ✅ | ✅ parent totals |

**Purchase Report and Accessory Purchase Report partition the purchase
sources between them** — every source is covered exactly once, and no row
appears in both. A *mixed* Purchase Voucher does appear in each, but as
different lines of itself, which is correct. Neither report should be
"completed" by adding the other's sources; the split is what keeps both
free of blank columns.

`Inventory Report` reads `Phone` and so covers every source implicitly —
a Phone is a Phone regardless of which intake created it. The five
sale-side reports never touched the purchase doctypes.

**Four intake doctypes coexist on purpose.** `Purchase Voucher` did not
replace the older three: their documents must stay openable and
cancellable, `Purchase Report`/`Supplier Report` still read `Purchase
Entry`, and `Purchase Entry.phone_created` is the only real `Link` field
to `Phone` in the entire schema. Same pattern as `Sales Entry` surviving
`Shop Sale`.

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
- **Do not pipe a multi-statement script into `bench console` via stdin.**
  It goes to IPython, which executes the input line-by-line as separate
  cells: nested function bodies get dedented and run at top level, and
  variables defined in one statement are missing from the next. The
  failure mode is deceptive — you get a wall of `NameError`s that look
  like genuine test failures on code that is actually fine (this cost two
  full test runs during the 2026-07-25 intake-cancel work before the cause
  was spotted). Wrapping everything in a single `def main(): ...` does NOT
  help; the indentation is already lost by then. For anything beyond a
  few flat statements, run a real script against the bench's own
  interpreter instead:
  ```
  cd ~/Documents/Work/mobile_shop/frappe-bench/sites
  ../env/bin/python myscript.py      # script does frappe.init(site=...)
                                     # + frappe.connect() ... frappe.destroy()
  ```
  Same caveat as a bench console session applies either way: neither
  shares a real request's commit/rollback wrapper, so test data written
  without an explicit `frappe.db.commit()` silently never persists.
- Two git repos exist: this inner one (`apps/mobile_shop/`) is the one that
  matters and has real commit history. An outer repo at
  `~/Documents/Work/mobile_shop/` is mostly unused — don't worry about it.
- Camera-scanning features require HTTPS or localhost (browser restriction).
  Use an ngrok tunnel for testing on a real device:
  `ngrok http 8000 --host-header="mobileshop.local:8000"`
