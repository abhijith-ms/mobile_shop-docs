# Second-Hand Mobile Shop ERP — Build Plan

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

## Phase 1 — Core Data Model

Build in this order. One DocType per Cline task.

### 1.1 Phone DocType
Fields: IMEI (unique), Brand, Model, Storage, RAM, Color, Battery Health %, Physical Condition, Display Condition, Accessories Included, Purchase Date, Purchase Price, Supplier (Link), Status (Select: In Stock / Reserved / Sold / Returned).

Server-side validation: reject duplicate IMEIs before save.

### 1.2 Supplier DocType
Fields: Name, Phone, Email, CPR/ID (optional), Notes.

### 1.3 Customer DocType
Fields: Name, Phone, Email, CPR (optional).

---

## Phase 2 — Purchase Workflow

Staff-facing form: scan/enter IMEI → select Supplier → enter Purchase Price, Brand, Model, Storage, Battery Health, Accessories → Save.

- Keyboard-wedge barcode scanners work as plain text input into the IMEI field — no special integration needed for v1.
- On save: creates a Phone record with Status = "In Stock".
- Camera/Bluetooth scanning: defer to a later phase, not required for MVP.

---

## Phase 3 — Sales Workflow + VAT Engine (compliance-critical)

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

### 4.1 Customer Invoice
Must show: Phone, IMEI, Total Price, and wording equivalent to "VAT Included – Profit Margin Scheme".
Must NOT show: Purchase Price, Profit, VAT amount, margin calculations.

### 4.2 Self-Billing Document (new requirement from NBR guide, not in original PRD)
When a phone is purchased from a non-VATable person (e.g. a private individual — your typical supplier case), NBR requires the dealer to self-issue a VAT invoice evidencing the acquisition, signed by the seller or their authorized signatory. Build a print format for this with a signature field, generated at Purchase save time.

---

## Phase 5 — Permissions

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
