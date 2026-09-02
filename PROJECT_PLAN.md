# Second-Hand Mobile Shop ERP — Build Plan

> **Note (2026-09-02):** The Phase 1–6 history below predates Shop Sale,
> Purchase Voucher, and everything built after "5 of 8 reports" — it stopped
> being actively maintained a while ago. `CLAUDE.md` in this same directory
> is the current, actively-maintained narrative for everything since. Active
> planning continues in the **"Accounting & POS Feature Set — Phase Plan
> (2026-09-02)"** section near the end of this file. This note is a minimum
> fix so a top-down read doesn't mistake the stale middle for current state;
> a full refresh of the old phase history is a separate, deferred job.

Custom Frappe/ERPNext app (`mobile_shop`) for a second-hand mobile phone shop in Bahrain, with IMEI traceability and Bahrain VAT Profit Margin Scheme (PMS) compliance.

**Tools:** Cline + Kimi 2.5 (via Bedrock) as primary build driver. Antigravity free tier for light/secondary tasks only (quota is unreliable — don't depend on it for the main build).

**Rule for every phase below:** hand Cline ONE numbered step at a time. After each step, open the site yourself and click through it before moving to the next. Don't let it build multiple DocTypes or features in a single task.

---

## Phase 0 — Environment Setup (do manually, not via Cline)

1. Install Frappe `bench` locally (Python, Node, MariaDB, Redis — the official bench install script handles most of this).
2. Create a new bench and a local site, install ERPNext, confirm login works.
3. Scaffold the custom app:
   ```
   bench new-app mobile_shop
   bench --site yoursite.local install-app mobile_shop
   ```
4. Create one dummy DocType through the UI to confirm the app registers correctly with the framework.

Do this step yourself so you understand the skeleton before handing anything to an AI agent.

---

## Phase 1 — Core Data Model ✅ DONE

Build in this order. One DocType per Cline task.

### 1.1 Phone DocType
Fields: IMEI (unique), Brand, Model, Storage, RAM, Color, Battery Health %, Physical Condition, Display Condition, Accessories Included, Purchase Date, Purchase Price, Supplier (Link), Status (Select: In Stock / Reserved / Sold / Returned).

Server-side validation: reject duplicate IMEIs before save.

### 1.2 Supplier — SKIPPED, using ERPNext core Supplier instead
Originally planned as a custom DocType, but ERPNext already ships a built-in "Supplier" DocType (module: Buying) with more functionality (address/contact management, purchase history, etc.) than the placeholder needed. Creating a duplicate custom DocType with the same name caused a naming collision that silently broke migration — do not recreate this. Phone.supplier links directly to core Supplier.

Note: hitting "New Supplier" the first time surfaced an unrelated ERPNext bug (`AttributeError: 'ERPNextAddress' object has no attribute 'is_your_company_address'`) caused by ERPNext's install completing before Redis was running, so its Address customizations never synced. Fixed via `frappe.reload_doctype("Address")` in the bench console, followed by a full migrate. Confirmed working after the fix.

### 1.3 Customer DocType
Fields: Name, Phone, Email, CPR (optional).

---

## Phase 2 — Purchase Workflow ✅ DONE

Note: naming series in purchase_entry.json needed dots around variable tokens — correct Frappe syntax is `PE-.YYYY.-.MM.-.#####`, not `PE-YYYY-MM-#####`. Also a reminder for all future doctypes: place under `apps/mobile_shop/mobile_shop/mobile_shop/doctype/<name>/` (inside the module subfolder), and double-check for naming collisions with ERPNext core doctypes (Supplier, Customer, Item, etc. all already exist in ERPNext core) before creating anything with a generic name.


Staff-facing form: scan/enter IMEI → select Supplier → enter Purchase Price, Brand, Model, Storage, Battery Health, Accessories → Save.

- Keyboard-wedge barcode scanners work as plain text input into the IMEI field — no special integration needed for v1.
- On save: creates a Phone record with Status = "In Stock".
- Camera/Bluetooth scanning: defer to a later phase, not required for MVP.

---

## Phase 3 — Sales Workflow + VAT Engine (compliance-critical) ✅ DONE

Verified end-to-end against NBR test case: purchase 200.000 BHD, sale 350.000 BHD → margin 150.000, VAT 13.636, net profit 136.364. Phone status flips to Sold on submit. Rejects re-selling a Sold phone and rejects unknown IMEIs cleanly.

Two fixes applied during build (worth remembering for future doctypes):
- `frappe.get_doc(doctype, filters)` raises DoesNotExistError rather than returning None — always check with `frappe.db.exists()` first if you need a clean error message instead of a raw traceback.
- Bahrain Dinar uses 3 decimal places (fils), not the Frappe Currency default of 2. Explicitly set `"precision": "3"` on any Currency field holding BHD amounts (selling_price, margin, vat_amount, net_profit, purchase_price — done on both Phone and Purchase Entry).


Staff-facing form: search/scan IMEI → select Customer → enter Selling Price → Save.

**Server-side calculation (per NBR Profit Margin Scheme guide):**
```
Margin = Selling Price - Purchase Price
If Margin <= 0: VAT = 0, Margin = 0  (no VAT when purchase price exceeds selling price)
VAT = Margin / 11
Net Profit (excl. VAT) = Margin / 1.1
```

On save: generate Invoice, flip Phone Status to "Sold".

**Verify by hand before trusting the code** — test against the NBR's own example:
Purchase 2,000 BHD → Sale 3,000 BHD → Margin 1,000 → VAT 90.909 → Net 909.091.

---

## Phase 4 — Invoice Print Format + Self-Billing Document

### 4.1 Customer Invoice ✅ DONE

Verified: shows Invoice Number, Date, Customer Name, Brand, Model, IMEI, Total Price, and "VAT Included - Profit Margin Scheme" footer. Confirmed purchase_price/margin/vat_amount/net_profit do NOT appear anywhere.

Note: when printing, make sure to select "PMS Customer Invoice" from the print format dropdown — Frappe defaults to the standard auto-generated format (which shows ALL fields including the restricted ones) unless you explicitly pick the custom one. Worth double-checking this every time until it's set as the default print format for this doctype.

Must show: Phone, IMEI, Total Price, and wording equivalent to "VAT Included – Profit Margin Scheme".
Must NOT show: Purchase Price, Profit, VAT amount, margin calculations.

### 4.2 Self-Billing Document (new requirement from NBR guide, not in original PRD) ✅ DONE

Verified: shows document reference, purchase date, supplier name, device details, purchase price, signature line, and NBR compliance paragraph. Note: this document DOES show purchase price (unlike the customer invoice) — that's correct and intentional, since this is an internal acquisition record, not something the end customer sees.

Bug hit + fixed: template initially referenced a "phone" field on Supplier which doesn't exist on ERPNext's core Supplier doctype — the correct field is "mobile_no" (populated from the linked primary contact). If it's blank, that's fine — the template just shows an empty value rather than erroring, as long as it's built with an `{% if %}` guard around it.

When a phone is purchased from a non-VATable person (e.g. a private individual — your typical supplier case), NBR requires the dealer to self-issue a VAT invoice evidencing the acquisition, signed by the seller or their authorized signatory. Build a print format for this with a signature field, generated at Purchase save time.

---

## Phase 5 — Permissions ✅ DONE

Verified with a real "Mobile Shop Staff" test user (not just reading JSON):
- Sales Entry: margin/vat_amount/net_profit correctly hidden (permlevel 1, no read for Staff).
- Phone: purchase_price correctly hidden (permlevel 1, no read for Staff).
- Delete blocked across all mobile_shop doctypes for Staff.
- Staff can create/submit Purchase Entry and Sales Entry normally.
- Admin/System Manager retain full visibility including all profit fields.

Bugs hit + fixed (important for any future doctype/permission work):
1. **Permlevel needs a matching level-1 permission row for every role**, not just `"permlevel": 1` on the field itself — each role needs both a level-0 row and a level-1 row in the doctype's `permissions` array, or restriction won't behave as expected.
2. **Link fields to doctypes with no explicit role grant will block the whole form.** Staff had zero permission on ERPNext core Supplier, which blocked the entire Purchase Entry form (not just the Supplier field) with a "Not permitted" error. Fixed via **Role Permissions Manager** (Ctrl+G → "Role Permissions Manager") — the correct, non-invasive way to grant a role access to a core doctype without touching ERPNext core files. Granted Mobile Shop Staff Read + Create on Supplier at level 0.
3. **`ignore_permissions=True` needed for automated cross-doctype writes.** When Purchase Entry's `on_submit` auto-creates a Phone record under a Staff session, `phone.insert()` silently drops any field the current user lacks write access to at its permlevel — so purchase_price was saving as 0 for Staff-submitted purchases. Fixed by using `phone.insert(ignore_permissions=True)` in `create_phone_record()`. This only bypasses permission checks, not `validate()` — the duplicate IMEI check still runs correctly.
4. Python controller (.py) changes sometimes need a full `bench start` restart (Ctrl+C then restart) to take effect, not just `clear-cache` — don't trust "no restart needed" claims at face value if a fix doesn't seem to apply.

---

# Post-MVP Addition: New/Unused Phone Sales (client request)

Client requested support for selling brand new/unused phones alongside second-hand ones. This matters for compliance, not just inventory — per the NBR guide (section 17.2), the Profit Margin Scheme only applies to used goods; new phones require standard VAT instead.

**Confirmed with client:**
- New/unused phone VAT rate: 10%.
- Staff select, per sale, whether the entered selling price is VAT-inclusive or VAT-exclusive (not hardcoded).

**Design:**
- `phone_type` (Select: Used/New, default Used) — added to Phone and Purchase Entry. ✅ DONE, verified.
- `vat_treatment` (Select: Inclusive/Exclusive) — to be added to Sales Entry, only relevant when phone_type = New.
- Calculation branches:
  - Used → unchanged PMS logic (margin ÷ 11), do not touch.
  - New + Inclusive → VAT = selling_price ÷ 11, net = selling_price ÷ 1.1, customer pays selling_price.
  - New + Exclusive → VAT = selling_price × 0.10, net = selling_price, customer pays selling_price × 1.10.
- A second print format, "Standard VAT Invoice" (showing VAT breakdown, as normally required for non-PMS sales), needed for New phone sales — the existing "PMS Customer Invoice" (no VAT shown) stays for Used phone sales only.

### ✅ DONE — verified by user directly in browser (not just Cline's self-report):
- New phone + Inclusive VAT: correct (vat_amount = price÷11, net_profit = price÷1.1, total_charged = price, margin = 0)
- New phone + Exclusive VAT: correct (vat_amount = price×0.10, net_profit = price, total_charged = price×1.10, margin = 0)
- Used phone: unchanged, matches original PMS math exactly (no regression)
- VAT Treatment field correctly hidden for Used, shown for New (depends_on)

Bug hit + fixed during build: a whitelisted method (`get_phone_type`) was initially defined nested inside the `SalesEntry(Document)` class without `self`, which breaks the module-level dotted path (`mobile_shop.mobile_shop.doctype.sales_entry.sales_entry.get_phone_type`) that the client script calls it by. Frappe whitelisted methods invoked via dotted path from JS must be standalone module-level functions, not class methods. Fixed by moving it outside the class.

**Still outstanding for this feature:** a second print format ("Standard VAT Invoice", showing the VAT breakdown) is needed for New-phone sales — the existing "PMS Customer Invoice" (no VAT shown) should remain reserved for Used-phone sales only. ✅ DONE.

Verified end-to-end by user, both roles:
- New phone + Inclusive: correct.
- New phone + Exclusive: Price (Net) 200.000, VAT (10%) 20.000, Total Charged 220.000 — confirmed correct on actual printed invoice.
- Confirmed working correctly for a Staff-logged-in user, not just Admin.
- Confirmed margin/vat_amount/net_profit remain hidden on the regular form view for Staff — the print-time bypass only affects the print output, not form-level access.

Bug hit + fixed: vat_amount/margin/net_profit are permlevel 1 (hidden from Staff), but this print format legitimately needs to SHOW vat_amount to the customer regardless of who's printing. Fixed by having the Jinja template fetch the value directly via `frappe.db.get_value(...)` inside the template rather than referencing `{{ doc.vat_amount }}` — a raw db lookup bypasses field-level permission checks, which is appropriate here since printing a required VAT figure to a customer isn't the same as exposing internal margin data to Staff.

Caution for future testing: when verifying Sales Entry behavior, always double check the Phone Type on the actual test record before trusting the result — an initial test accidentally ran against a Used phone while intending to test the New-phone branch, which gave misleadingly-plausible-looking (but wrong-branch) numbers.

Minor known issue (not yet investigated): one test invoice showed Brand/Model as "None" — worth checking whether that Phone record was just missing test data, or whether the print format's IMEI→Phone lookup has an edge case worth checking.

---

**Post-MVP New/Unused Phone feature is now fully complete**: phone_type tracking, branching VAT calculation (PMS margin for Used, standard 10% VAT for New with Inclusive/Exclusive options), and both compliant print formats.



---

# Post-MVP Addition: Camera-Based IMEI Scanning (client request)

Added camera-based barcode/QR scanning (html5-qrcode library) as "Scan IMEI" button on Purchase Entry and Sales Entry, as a convenience alongside manual typing and USB/Bluetooth scanners (which already worked automatically via keyboard-wedge input, no code needed).

### ✅ DONE — all issues found and fixed:
1. **Button initially invisible / grouped under overflow "..." menu**: `frm.add_custom_button`'s third param groups buttons; removed to make it standalone. Also required using the `refresh` event (not `onload`) to reliably fire on new/unsaved documents.
2. **Files accidentally deleted mid-session** by an overly-broad `rm -rf` from Cline. Recovered cleanly via `git restore` since the repo was git-tracked and files were previously committed. **Lesson: commit working states regularly**, and never let an agent run `rm -rf` near `apps/*/public/` source directories — only `sites/assets/` build output is safe to clean.
3. **Camera wouldn't restart on 2nd/3rd scan attempt** (blank video, permission prompt fired but no feed) — root cause was TWO compounding issues:
   - Reusing the same static DOM element ID (`imei-qr-scanner`) across sessions caused browser-level video rendering to break even though html5-qrcode's internal state reported "SCANNING" successfully. Fixed by generating a unique element ID per scanner session (`imei-qr-scanner-` + timestamp).
   - The dialog's CSS layout/animation wasn't finished before `start()` was called on later attempts, so the target div had zero dimensions at call time. Fixed with a `waitForElementAndStart()` polling loop (via `requestAnimationFrame`) that waits for real `offsetWidth`/`offsetHeight` before starting the camera, rather than a fixed delay.
   - Confirmed via detailed console logging (DOM state, instance IDs, post-start `<video>` element inspection) — guessing at fixes without this level of tracing burned several rounds before the real cause was found.
4. **Luhn checksum validation**: added as a SOFT warning only (frappe.msgprint, orange indicator), not a hard block — staff must always be able to save with a non-standard IMEI (real-world second-hand inventory won't always be pristine). Only length/format (14-16 digits) is a hard reject. Applies uniformly via a shared `mobile_shop/utils/imei.py` module, called from Phone, Purchase Entry, AND Sales Entry (initially missed on Sales Entry — a 13-digit EAN saved successfully there until this gap was caught and fixed).
5. IMEI-SV (16-digit) values correctly skip Luhn checking entirely, since that format doesn't carry a real check digit.

Confirmed by user directly in browser: scan → clear → rescan → rescan again, all working; hard-reject on invalid length confirmed on Sales Entry; soft warning on bad checksum confirmed allowing save.

---

# Post-MVP Addition: Simplified Workspace UI (client feedback)

Client saw the raw ERPNext sidebar (full module list) and asked for something simpler and more focused. Addressed in two parts:

1. **Hid all unrelated ERPNext modules** (Accounting, Buying, Selling, Stock, Assets, Manufacturing, Quality, Projects, Support, Website, CRM, Tools, ERPNext Settings, Integrations, Build) from the sidebar via Workspace visibility settings — done manually by user, not code.

2. **Redesigned the "Mobile Shop" workspace itself** into a proper dashboard: ✅ DONE
   - "Overview" section with 3 live Number Cards: Phones In Stock (filtered count, status="In Stock"), Sales This Month, Purchases This Month (both using Timespan filters on their date fields).
   - "Daily Tasks" section: Sales Entry / Purchase Entry shortcuts.
   - "Records" section: Phone / Customer / Supplier shortcuts.
   - Verified: Timespan "this month" filters genuinely apply correctly (confirmed via manual date-range query matching the card's live count exactly), not just coincidentally equal to the total.

Bugs hit + fixed during this round:
1. `bench --site <site> export-fixtures` reported success and Number Card *records* were created correctly, but the Workspace's own `content` field (which defines layout/blocks) was NOT updated by the same export — required directly setting `ws.content` via `frappe.get_doc("Workspace", ...)` + `.save()` in the bench console instead of trusting fixture export/import for this specific field.
2. Workspace content JSON block schema (for "header" and "number_card" block types) had to be reverse-engineered by inspecting a working built-in workspace's actual stored `content` field (via `frappe.db.get_value("Workspace", "Home", "content")`) rather than guessed — guessing produced blocks that silently failed to render with no error.
3. Number Cards showing `show_percentage_stats: 1` triggered a genuine bug in Frappe core itself (`get_percentage_difference` → `TypeError: 'NoneType' object is not callable`) — worked around by setting `show_percentage_stats: 0` on all cards, since a simple count doesn't need a trend comparison anyway.
4. Initial Timespan filter value (`"next 30 days"`) was wrong direction (future dates, not current month) and also malformed structure — correct syntax is `[["<Doctype>", "<date_fieldname>", "Timespan", "this month"]]`.

**Lesson reinforced**: for Frappe UI/config-as-data features (workspaces, number cards, print formats), "the export/fixture command succeeded" and "the correct schema was used" are separate claims — verify both against a known-working example in the same Frappe version rather than trusting either in isolation.

** Inventory tracking with IMEI uniqueness, purchase workflow, PMS-compliant sales/VAT engine, both required print formats, and role-based permissions are all built and verified. Remaining: Phase 6 (dashboard/reports) — lower priority since it only reads data the earlier phases already produce correctly.


- Staff role: can create Purchase and Sales records, cannot see Purchase Price / profit fields on Sales, cannot delete records, cannot touch VAT Settings, cannot manage users.
- Admin role: full access to everything, including profit/margin reports.

---

## Phase 6 — Dashboard + Reports (build last)

Only after Phases 1–5 are solid, since these just read data the earlier phases already produce correctly.

- Dashboard: Inventory Count, Inventory Value, Today's Sales, Monthly Sales, Total Revenue, Purchase Cost, Profit Margin, VAT Payable, Net Profit, Pending Supplier Settlements, Phones Sold, Top Selling Brands, Low Stock Alerts.
- Reports: Inventory, Sales, Purchase, Supplier, Customer, Profit, VAT, IMEI History.

---

## Compliance Checklist (do not skip)

- [ ] Obtain NBR approval for using the Profit Margin Scheme before going live.
- [ ] Verify final invoice wording with an accountant/NBR contact — "no VAT amount shown" is a legal requirement, not a style choice.
- [ ] Confirm self-billing document/signature process for private-individual supplier purchases.
- [ ] Do not use this system for real customer transactions until the above are signed off.

---

## Hosting (decide after Phase 3 works, not before)

- **Build phase:** local Frappe bench, no hosting cost.
- **Go-live options:**
  - VPS (DigitalOcean/Hetzner/regional GCC provider), ~$10–$40/month, full control, you own patching/backups.
  - Frappe Cloud private bench, ~$25/month minimum for custom app support with SSH access (the $5 shared-site tier does NOT support custom apps).
- Check whether NBR/Bahrain data-handling expectations push you toward a GCC-region host before committing.

---

# Cline / Kimi 2.5 Task Prompts

Copy one prompt per session. Wait for it to finish and verify in the UI before pasting the next.

## Prompt — Phase 1.1: Phone DocType

```
We are building a custom Frappe app called `mobile_shop` inside an existing bench (app already scaffolded and installed on the site). Do not modify ERPNext core.

Create a new DocType called "Phone" in the mobile_shop app with these fields:
- imei (Data, required, unique)
- brand (Data)
- model (Data)
- storage (Data)
- ram (Data)
- color (Data)
- battery_health (Percent)
- physical_condition (Select: Excellent, Good, Fair, Poor)
- display_condition (Select: Excellent, Good, Fair, Poor)
- accessories_included (Small Text)
- purchase_date (Date)
- purchase_price (Currency)
- supplier (Link to Supplier doctype — create a placeholder Supplier doctype if it doesn't exist yet, with just a "supplier_name" field for now)
- status (Select: In Stock, Reserved, Sold, Returned; default "In Stock")

Add a server-side validation in the Phone doctype's Python controller that raises a frappe.ValidationError if a Phone with the same IMEI already exists (excluding the current document on update).

Follow standard Frappe conventions (doctype JSON + controller .py file in the correct module folder). Do not add any UI beyond what Frappe generates automatically from the DocType definition. After creating it, tell me the exact bench commands I need to run to migrate and see it in the UI.
```

## Prompt — Phase 1.2: Supplier DocType (skip if already created above)

```
In the mobile_shop app, create a full "Supplier" DocType with fields:
- supplier_name (Data, required)
- phone (Data)
- email (Data)
- cpr_id (Data, optional)
- notes (Small Text)

If a placeholder Supplier doctype already exists from the Phone doctype task, extend it with these fields instead of creating a new one. Tell me the migrate command to run afterward.
```

## Prompt — Phase 1.3: Customer DocType

```
In the mobile_shop app, create a "Customer" DocType with fields:
- customer_name (Data, required)
- phone (Data)
- email (Data)
- cpr (Data, optional)

Follow the same conventions as the Phone and Supplier doctypes already in this app. Tell me the migrate command to run afterward.
```

## Prompt — Phase 2: Purchase Workflow

```
In the mobile_shop app, we need a Purchase entry workflow. Create a "Purchase Entry" DocType with fields:
- imei (Data, required — this is scanned/typed by staff)
- supplier (Link to Supplier, required)
- purchase_price (Currency, required)
- brand, model, storage, battery_health (same types as in the Phone doctype)
- accessories_included (Small Text)
- purchase_date (Date, default today)

On submit of a Purchase Entry, write a server-side controller hook that:
1. Checks no Phone with this IMEI already exists (reuse the validation logic from the Phone doctype if possible).
2. Creates a new Phone record using the data entered, with status "In Stock".
3. Links the Purchase Entry to the created Phone record for traceability.

Show me the controller code before I run migrate, so I can review the logic.
```

## Prompt — Phase 3: Sales Workflow + VAT Engine

```
In the mobile_shop app, create a "Sales Entry" DocType with fields:
- imei (Data, required — staff searches/scans this to find the Phone record)
- customer (Link to Customer, required)
- selling_price (Currency, required)
- sale_date (Date, default today)
- invoice_number (Data, auto-generated or Frappe naming series)

Server-side controller logic on submit:
1. Look up the Phone record by IMEI. If not found or status is not "In Stock", raise a validation error.
2. Calculate:
   margin = selling_price - phone.purchase_price
   if margin <= 0: margin = 0, vat = 0
   else: vat = margin / 11
   net_profit = margin / 1.1
3. Store margin, vat, and net_profit as fields on the Sales Entry (these are internal-only, not shown on the customer invoice).
4. Update the linked Phone record's status to "Sold".

Write these as fields on the doctype: margin, vat_amount, net_profit (all Currency, read-only, calculated by the server — never editable by the user).

After building this, show me the calculation code specifically, so I can manually verify it against this test case: purchase_price = 2000, selling_price = 3000 should give margin = 1000, vat_amount = 90.909, net_profit = 909.091.
```

## Prompt — Phase 4.1: Customer Invoice Print Format

```
In the mobile_shop app, create a custom Print Format for the "Sales Entry" doctype called "PMS Customer Invoice" that shows only:
- Phone (brand, model)
- IMEI
- Customer name
- Selling Price (labeled "Total Price")
- Invoice number and sale date
- Footer text: "VAT Included - Profit Margin Scheme"

It must NOT display: purchase_price, margin, vat_amount, or net_profit anywhere on this print format, even if those fields exist on the underlying doctype. Double check the template doesn't accidentally pull them in.
```

## Prompt — Phase 4.2: Self-Billing Document

```
In the mobile_shop app, create a custom Print Format for the "Purchase Entry" doctype called "Self-Billed Purchase Invoice", used when the supplier is a private individual (non-VATable person). It should show:
- Supplier name, phone
- Phone brand/model/IMEI
- Purchase price
- Purchase date
- A signature line labeled "Signature of Seller / Authorized Signatory"
- Text noting this document is self-issued by the dealer to evidence acquisition from a non-VATable person, per Bahrain NBR Profit Margin Scheme documentation requirements.
```

## Prompt — Phase 5: Permissions

```
In the mobile_shop app, set up Frappe role permissions for a "Mobile Shop Staff" role:
- Can create and read Purchase Entry and Sales Entry documents.
- Cannot delete any mobile_shop doctype records.
- On the Sales Entry doctype, hide/restrict read access to the margin, vat_amount, and net_profit fields for this role (use field-level permissions or permlevel).
- Cannot access a "VAT Settings" doctype (create a simple one if it doesn't exist yet, just to lock it down).
- Cannot access User management.

Leave a "Mobile Shop Admin" role (or use existing System Manager) with full access to everything. Show me the permission configuration before I apply it.
```

---

*Keep this file in your project root and update the checkboxes/status as each phase completes.*

---

# Critical Frappe Lesson: "Standard" Doctypes, Fixtures, and Migrate Behavior

Discovered the hard way across the Workspace redesign AND the Number Card dashboard — worth reading before touching either again, or before building any other Workspace/Number Card/similar "standard" doctype feature.

**The problem:** Workspace and Number Card are both `is_standard: 1` doctypes. This means:
1. They auto-sync from their own **module-level JSON files** (e.g. `mobile_shop/mobile_shop/workspace/mobile_shop/mobile_shop.json`, `mobile_shop/mobile_shop/number_card/<name>/<name>.json`) during `bench migrate`.
2. **BUT this sync is INSERT-ONLY** — it creates the record if it doesn't exist, but does NOT update an already-existing record, no matter how many times you edit the file and re-migrate. (Confirmed by testing directly: editing the file, migrating, and checking the DB value — it never changed until set via `frappe.db.set_value()` in the console.)
3. **If the doctype is ALSO listed in `hooks.py`'s `fixtures = [...]`**, this creates a SECOND, independent sync mechanism using a totally different file path (`mobile_shop/fixtures/<doctype>.json`), which CAN overwrite existing records with stale content on every migrate. This was the actual cause of the workspace/number-card content repeatedly reverting to old versions — stale `fixtures/workspace.json` and `fixtures/number_card.json` files (the latter full of ERPNext's own default cards from Manufacturing/Assets/CRM, unrelated to this app) kept re-importing over the real data.

**The permanent fix applied:**
1. Removed `"Workspace"` and `"Number Card"` from `fixtures` in `hooks.py` entirely.
2. Deleted the stale `mobile_shop/fixtures/workspace.json` and `mobile_shop/fixtures/number_card.json` files.
3. For any ALREADY-EXISTING record needing correction, fix it directly via `frappe.db.set_value(...)` + `frappe.db.commit()` in the bench console — editing the module JSON file alone does NOT fix an existing record, only future fresh installs.
4. Always verify a fix survives by running `bench migrate` at least twice in a row and refreshing the browser each time — one successful migrate is not proof the fix is permanent.

**Specific bugs fixed under this pattern:**
- Workspace `content` kept reverting to the old plain-shortcut-list layout.
- Number Card `show_percentage_stats` kept reverting to `1`, triggering a genuine Frappe CORE bug (`get_percentage_difference` → `TypeError: 'NoneType' object is not callable`) whenever a card tried to show a trend comparison. Fixed by setting to `0`.
- Number Card `filters_json` kept reverting to an invalid flat-dict format instead of the correct list-of-lists format: `[["Sales Entry", "sale_date", "Timespan", "this month"]]`.

**Script Report folder naming is NOT arbitrary.** Frappe derives the expected Python module path automatically from the Report's `name` field (scrubbed: lowercased, spaces→underscores). A report named "IMEI History Report" MUST live in folder `imei_history_report` with file `imei_history_report.py` — not a shorter name like `imei_history`, even if the JSON's `"name"` field is correct. Wrong folder name → `ModuleNotFoundError` at runtime.

**Recovery technique that worked:** when unsure whether current file content matches a previously-verified version, check `git log`/`git show` against a known-good commit rather than reconstructing from memory or trusting an AI agent's summary of "what changed."

---

# Phase 6 — Reports ✅ PARTIALLY DONE (5 of 8)

Five reports built, individually schema-verified (via real `frappe.get_meta(...).get_fieldnames()` output, never assumed), and confirmed working in the browser as both Staff and Admin:

1. **IMEI History Report** — full lifecycle by IMEI. Admin-only: Purchase Price.
2. **Sales Report** — filterable by date/customer/IMEI/brand/model/VAT treatment. Admin-only: Purchase Price, Margin, VAT Amount, Net Profit. Staff-visible: Total Charged.
3. **Purchase Report** — filterable by date/supplier/IMEI/brand/phone type. ~~Admin-only: Purchase Price.~~ **Staff-visible since 2026-07-29** — it reads `purchase_price` from the intake doctypes, where it is permlevel 0, not from `Phone` where it is permlevel 1. (IMEI History Report and Sales Report above DO read `Phone.purchase_price` and correctly remain Admin-only.)
4. **Customer Report** — purchase count and total spent, COALESCE-safe for zero-sales customers. Same for both roles.
5. **Supplier Report** — purchase count and total purchase value. Admin-only: Total Purchase Value. Admin-only "Include Disabled Suppliers" filter.

**Consistent pattern used across all five** (deviating from this caused every bug hit along the way):
- `has_admin_role()`: `bool(set(frappe.get_roles(frappe.session.user)) & {"Mobile Shop Admin", "System Manager"})`.
- Sensitive fields (purchase_price, margin, vat_amount, net_profit) OMITTED ENTIRELY from both the `columns` list AND the SQL SELECT when not admin — never fetched, never a NULL placeholder. A `"permlevel": 1` tag on a Script Report column does NOT enforce anything by itself — Script Reports run raw SQL directly, bypassing document-level field permissions.
- Column lists built via `.append()` + `",\n".join(...)`, never manual comma concatenation (caused a real double-comma SQL bug once).

**Two/three reports were built with entirely FABRICATED schema fields in a rushed, unreviewed batch and deleted**: Inventory Report, Profit Report, VAT Report (plus an early broken IMEI History variant). Referenced non-existent child tables (`tabPurchase Entry Item`, `tabSales Entry Item` — this app has NO child tables) and invented fields (`grade`, `posting_date`, `customer_name`, `cr_number`, `mobile_number`, "unpaid amounts", etc.). **Lesson: always run `frappe.get_meta(doctype).get_fieldnames()` and show the REAL output before writing report code — never batch-build multiple reports without reviewing each one individually first.**

**Workspace integration**: All five reports added as Shortcut blocks under a "Reports" header, built using Frappe's visual Workspace Editor (not hand-written JSON — repeated hand-written JSON attempts produced blank/broken pages). When adding a Shortcut to a Report, its "Type" field must be explicitly set to "Report" — defaults to "DocType" and won't show reports in the picker otherwise.

**Remaining, not yet built**: Inventory Report, Profit Report, VAT Report — needed per the original PRD, must be rebuilt from scratch with proper schema verification and one-at-a-time review, exact same pattern as the five working reports above.

---

# Accounting & POS Feature Set — Phase Plan (2026-09-02)

Everything below reflects a real client meeting with the shop's accountant,
held ahead of this phase. He is satisfied with the overall setup, flagged
that the Desk UI should eventually change (matching the already-planned
custom React frontend), and gave ten feature requests. Do not relitigate the
architecture decision below without new information from the client.

## Accountant meeting — outcome

### The ten requested items

1. **Sale price when adding phones.** A sale price captured at intake.
2. **POS: show VAT when Exclusive is selected.** Currently VAT only appears
   on the printed invoice, not live on screen.
3. **"Recent" option in POS.**
4. **POS payment mode.** Method list is Cash, Card, Benefit Pay, Credit,
   Mixed (Benefit Pay is Bahrain's national payment system).
5. **Receivable / Payable shown on the home page.**
6. **Payment Voucher after Purchase Voucher.** Records money actually paid
   to a supplier against a Purchase Voucher.
7. **Purchase Report should show VAT.**
8. **Daybook.**
9. **Cash Book and Bank Book.**
10. **Credit / partial-payment sales.** A customer may not pay the full
    amount at time of sale; the shortfall is tracked as an outstanding
    balance and reflected in the Daybook.

### Answers received on follow-up questions

- **Payment methods** — Cash, Card, Benefit Pay, Credit, Mixed. "Credit"
  means no money moved at time of sale. "Mixed" means any combination of
  the others.
- **Split payments** — each method's amount must be recorded separately,
  not just a total.
- **Banks** — bank names exist so the shop can track which bank the money
  went into. A bank must be selectable on any method where money lands in a
  bank (Card, Benefit Pay). Banks are an admin-managed list — admin adds
  and removes accounts, with a default pre-selected.
- **Credit limits** — not needed in the system. The shop handles that
  themselves. The Credit option is for rare cases only, not routine use.
- **Who can give credit** — Staff can give credit. Not admin-only.
- **Credit sale customer** — a real customer is mandatory. Walk-in is not
  acceptable for a credit sale.
- **Balance payment** — recorded at the time it is actually paid. This
  means a separate customer receipt entry is required (mirroring Payment
  Voucher, but for money in). This was a gap in the accountant's original
  list.
- **Payment Voucher scope** — new Purchase Vouchers only. The three old
  intake doctypes are being retired; no payments needed against them.
- **Sale price at intake** — a suggested price, changeable at the counter.
- **Purchase Report VAT** — yes, as separate net, VAT and gross columns.
- **Register visibility** — Daybook, Cash Book and Bank Book are visible to
  Staff, not admin-only.
- **Register format** — no sample provided; use a standard/conventional
  layout.

> Staff visibility on the registers runs against the Profit/VAT Report
> precedent (those stay Staff-blocked). It is consistent with the earlier
> deliberate decision to show `purchase_price` to Staff in the purchase-side
> reports (see Hard Rule 3 in CLAUDE.md), but worth a sanity check on what
> the registers actually expose before building them — a Daybook or Bank
> Book is closer to a full financial statement than a purchase report is.

## Architecture decision — SETTLED

**Fully bespoke, designed GL-ready.** The accounting layer will NOT use
ERPNext's native `Bank Account`, `Mode of Payment`, or `Payment Entry`.

### Reasoning (do not relitigate without new information)

- Native `Bank Account` is not standalone. It links to a `Bank` doctype and,
  to be accounting-useful, to an `Account` in the Chart of Accounts — which
  drags in Company and CoA setup this app has deliberately avoided.
- Every native doctype Staff must see in a Link field requires Custom
  DocPerm rows on a core doctype. That is the Hard Rule 15 trap in
  CLAUDE.md, which already cost a full cycle on Item and UOM.
- The main payoff of native doctypes is free Desk UI and free native
  reports. The planned custom React frontend and the bespoke report layer
  make both irrelevant here.
- The bank master is a handful of rows — trivially migratable in Phase A.

### What is being built instead

- A bespoke `Bank` master: name, account number, `is_default`, `disabled`.
- A bespoke payment child table on Shop Sale and on Payment Voucher.

### The GL-ready field contract

See CLAUDE.md Hard Rule 20 for the full field contract every payment row
must carry. Summary: date, direction (in/out), method, amount, bank link,
party, and the parent document it settles, using ERPNext Payment Entry's own
field vocabulary (`mode_of_payment`, `paid_amount`, `party`, `party_type`,
`reference_doctype`, `reference_name`) so a future Phase A GL migration is
mechanical rather than a re-design.

**Accepted trade-off:** Phase A will need a backfill job to produce
historical GL entries, rather than having them accrue natively from day one.

## Build sequencing — Phases 1 to 6

Dependency-ordered. Each phase follows the established working style: scope
→ plan → review → small reviewed steps with one commit each → empirical
verification → browser pass as both a real Staff account and Mobile Shop
Admin, never Administrator.

### Phase 1 — quick wins, no dependencies

Items 1, 2, 3, 7. Build order within the phase: **1a first** (smallest).
**Phase 1 is fully done as of 2026-09-02** — all four items built and
verified; see CLAUDE.md's "Current state" section for each item's build
and verification detail.

- **1a — Purchase Report VAT columns (item 7). Done, commit `21839d6`.** Add `net_amount`,
  `vat_amount`, `amount`, following exactly the pattern already built in
  Accessory Purchase Report, including the NULL-not-zero rule for sources
  that never recorded VAT. Purchase Voucher lines carry per-line VAT; the
  legacy sources do not.
- **1b — Suggested sale price at intake (item 1). Done, commit `30b19ce`.** Field goes on both
  `Phone` and the Purchase Voucher phone line. Suggested, not fixed: POS
  pre-fills from it and the cashier can change it at the counter with no
  special permission. Open in its own plan: how the value flows from
  voucher line to Phone on submit; what happens for batch-received phones
  where the Phone record does not exist until sale time; and what permlevel
  the field gets — decide deliberately rather than copying, since
  `purchase_price` is permlevel 1 on `Phone` but permlevel 0 on the intake
  doctypes, and sale price is not margin-sensitive the way purchase price
  is.
- **1c — Live VAT display in POS on Exclusive (item 2). Done, commit `c7a46ed`.** Display-only. The
  calculation already exists and is verified correct — reuse it, do not
  reimplement.
- **1d — Quick-select product panel in POS (item 3). Done, commit `a47ea45`.** Scope corrected
  2026-09-02: this is NOT a recent-sales log (that was the original
  meeting framing) — it is a quick-select panel of top-selling *products*
  so staff can add common items to the cart fast, ranked by frequency over
  a rolling 7-day window and filtered by sellable stock. 5 entries,
  always-visible panel (not a dialog), products only (no time/customer/
  total columns). Accessories are the priority case; phones are included
  by model (not unit — units are IMEI-unique), and tapping a phone-model
  tile prompts for IMEI selection before adding to cart, a visibly
  different tap behavior from an accessory tile's straight-to-cart add.
  Confirmed by reading the code: the void-last-sale feature's recent-sale
  state (`this.last_sale`) is a single completed *sale document*, not a
  product ranking, so this is a separate lookup, not a reuse or extension
  of that state. Full design, including where the ranking query lives, its
  per-load cost, how sellable stock is determined for both accessories and
  batch-tracked phone models, and the coexistence with the existing
  all-time "Browse Accessories" panel, at
  `~/.claude/plans/quick-select-product-panel-pos.md`.

### Phase 2 — bespoke `Bank` master

Small and standalone. Unblocks item 4, item 6, and the Bank Book.

### Phase 3 — money in (items 4 + 10)

Built together — they are one form section, not two.

- Needs a child table, one row per payment component (method, amount,
  bank), because Mixed can span Card and Benefit Pay landing in different
  banks. A few fields on Shop Sale cannot represent that.
- A real customer is forced on any Credit component.
- BHD 3-decimal rounding rule applies: when rounding a total from multiple
  parts, round two and derive the third.
- Includes the customer receipt document for later balance payments.

### Phase 4 — money out (item 6)

Payment Voucher against Purchase Voucher. Reuses Phase 3's payment
component rather than reimplementing it. Purchase Voucher only — old intake
doctypes are out of scope.

### Phase 5 — the registers (items 8, 9)

Daybook, Cash Book, Bank Book. Pure read layers over Phases 3 and 4 — they
cannot be built earlier because the data does not exist yet. Each payment
child row is already a register line. Cash Book and Bank Book are the same
query filtered by method, with running balances.

### Phase 6 — Receivable / Payable on home (item 5)

Last, deliberately. Receivable is only meaningful once item 10 exists;
Payable only once item 6 does. Building earlier would ship a dashboard tile
showing two permanent zeros.

## Open question — not decided yet

**Frontend vs. backend sequencing.** Phases 3–6 build Desk-side forms that
the planned custom React frontend would then need to re-implement. Whether
the accounting layer or the frontend rebuild goes first is unresolved.
Record it as open — do not assume Desk is the permanent target.