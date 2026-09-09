# Mobile Shop ERP — End-to-End Testing Guide

A checklist for manually testing every real flow in the app before treating it as
ready for the shop to use daily. Work through it section by section; check each
box as you confirm it. Where a step says **(Staff)** or **(Admin)**, log in as
that role first — see [Accounts](#accounts) below.

Leave a note next to anything that doesn't behave as described here — that's
either a real bug, or this guide is out of date and needs correcting.

---

## Accounts

Two real, non-superuser accounts exist for testing. Never use the `Administrator`
superuser account to test anything permission-related — it bypasses every
permission check, so it will hide real bugs.

| Role | Account | Password |
|---|---|---|
| Mobile Shop Staff | `clashams4@gmail.com` | (yours) |
| Mobile Shop Admin | `msadmin.test@mobileshop.local` | (yours) |

## URLs

- App: `http://mobileshop.local:8000/shop/` (redirects to login if you're not
  signed in)
- Login page: `http://mobileshop.local:8000/login`
- Older Desk admin area (Records, User management fallback): `http://mobileshop.local:8000/app`

## Before you start

- Camera IMEI scanning needs either `localhost` or HTTPS to work in the browser
  — if testing on a phone/tablet over plain HTTP on the local network, the
  camera button will show "Camera Unavailable" instead of opening. That's
  expected in that setup, not a bug (see [§13](#13-camera-scanner-hardware)).
- Everything in this guide creates real records. Where a step says **clean up**,
  do it before moving to the next section, so later steps aren't testing against
  leftover data. If you'd rather not clean up as you go, that's fine too — just
  expect stock counts, reports, and dashboard stats to reflect whatever you've
  left behind.
- BHD amounts should always show **3 decimal places** (fils), everywhere —
  cart totals, reports, invoices, receipts. If you see 2 decimals anywhere,
  flag it.

---

## 1. Login & Session

- [ ] Visit `/shop/` while signed out → lands on a branded login form (not a
      bare Frappe page).
- [ ] Wrong password → a clean inline error message, not a raw error page.
- [ ] Correct password (either account) → lands on the Dashboard.
- [ ] Visit `/shop/pos` directly while signed out → redirected to login, and
      after logging in you land back on `/shop/pos` (not the dashboard) —
      confirms the redirect-back-after-login works for a deep link, not just
      the homepage.
- [ ] Top bar shows your name and role, with a **Log out** button, on every
      page (Dashboard, POS, reports, everywhere).
- [ ] Log out → back at the login form. Confirm you can't navigate back into
      the app via the browser back button without logging in again.
- [ ] Log in as the other account → confirms account-switching works cleanly
      (logout, then login as the other one — there's no separate "switch
      user" shortcut, this is the intended flow).

---

## 2. Staff Management (Admin only)

- [ ] **(Staff)** Confirm the **Manage Staff** tile does *not* appear on the
      Dashboard's Records section at all.
- [ ] **(Staff)** Confirm navigating directly to `/shop/staff` bounces you back
      to the Dashboard.
- [ ] **(Admin)** Manage Staff tile is visible, opens the staff list.
- [ ] **(Admin)** List shows every real staff/admin account, their role, and
      enabled/disabled status.
- [ ] **+ Add Staff**: fill in a real test name/email, a password (try
      something simple like `test1` — should be accepted, no strength
      complaints), pick a role, submit. New account appears in the list
      immediately.
- [ ] Log in as the new test account in a different browser/incognito window
      — confirms the password you set actually works and the role you picked
      actually applies (Staff-role account should NOT see Admin-only tiles;
      see [§9](#9-reports)).
- [ ] Back as Admin: **Edit** the test account, change its role to the other
      one, save. Confirm the list now shows the new role.
- [ ] **Edit** again, set a new password, leave role as-is, save. Log in as
      that account with the *new* password to confirm it changed.
- [ ] **Disable** the test account. Try logging in as it — should be blocked.
- [ ] **Enable** it again — login should work again.
- [ ] Confirm your *own* row (the Admin account you're logged in as) has no
      Edit/Disable buttons at all — just says "You". This is deliberate,
      so an Admin can't accidentally lock themselves out.
- [ ] **Clean up**: disable or leave the test account as you prefer — there's
      no delete option in-app by design (disable, don't delete).

---

## 3. Dashboard

- [ ] **(Staff)** and **(Admin)**: confirm the 5 stat tiles at the top
      (Phones In Stock, Purchases This Month, Sales This Month, Total
      Receivable, Total Payable) show sensible numbers, all money figures to
      3 decimals.
- [ ] **(Staff)**: confirm **Profit Report** and **VAT Report** tiles are
      absent from the Reports section.
- [ ] **(Admin)**: confirm all 12 report tiles are present (see full list in
      [§9](#9-reports)).
- [ ] Click one tile of each type — a Page tile (Point of Sale), a DocType
      tile (Purchase Voucher), a Report tile (any report) — confirm each
      navigates correctly and stays inside the app (no bounce out to Desk).

---

## 4. Purchase Voucher — Intake

All of these start from the **Purchase Voucher** tile → `+ Add Line`.

### 4.1 New phone, IMEI captured immediately

- [ ] Add a Phone Line: brand/model text, quantity, purchase price, VAT
      Treatment = Inclusive, a supplier.
- [ ] Use **Capture IMEIs** on the line — scan or type a real-looking 15-digit
      IMEI for each unit. Progress counter updates live ("1 / 1 IMEIs added").
- [ ] Try an IMEI that fails the Luhn checksum (e.g. repeat digits like
      `111111111111111`) — should show a soft warning but still let you save
      (warn, not hard-block).
- [ ] Try an IMEI that's the wrong length — should hard-reject.
- [ ] Save the voucher (still a draft) — reload the page, confirm the line
      and captured IMEI(s) are still there (proves the save round-trip
      works, not just the in-memory state).
- [ ] Submit. Confirm a real `Phone` record now exists at `In Stock` for
      that IMEI (check via Inventory Report, or Desk's Phone list).

### 4.2 Used phone, IMEI captured immediately

- [ ] Same as above but VAT Treatment left blank/Used-phone path (no VAT
      Treatment on Used lines — confirm the field is hidden or ignored for
      Used).
- [ ] Confirm a Used line **cannot** be left partially captured — the box
      UPC/remainder path is blocked for Used lines (per the compliance rule:
      Used stock must always be individually tracked).

### 4.3 Batch / untracked New phone (box of units, no per-unit IMEI yet)

- [ ] Add a Phone Line for a New phone, enter a quantity larger than the
      number of IMEIs you capture (e.g. qty 5, capture 2 IMEIs).
- [ ] Enter a Box UPC for the remainder — required for this to save.
- [ ] Submit. Confirm: 2 real tracked `Phone` records exist, and the
      remaining 3 units show up as untracked `Phone Batch` stock against
      that UPC (Inventory Report or Phone Batch list).
- [ ] Try the same thing with a **Used** phone line and no UPC entered for
      a leftover remainder — should hard-block the save (Used units can't
      go untracked, per §4.2's rule).

### 4.4 Accessory line

- [ ] Add an Accessory Line: pick or create an Item, quantity, price, VAT
      Treatment.
- [ ] Submit. Confirm `Item.current_stock` increased by the quantity you
      entered (check the Item, or just note it and confirm the number goes
      down correctly later in §5.4).

### 4.5 Mixed voucher (phones + accessories, mixed VAT treatments)

- [ ] One voucher with a New phone line (Exclusive VAT), a Used phone line,
      and an Accessory line (Inclusive VAT) — all in one document.
- [ ] Submit successfully. This is a **soft warning, not a hard block** — you
      should see a warning about mixing PMS/standard-VAT lines but still be
      able to submit. (If you want this to become a hard block instead,
      that's a real product decision — flag it, don't just note it as a bug.)

### 4.6 Payment at intake

- [ ] On any of the vouchers above, before submitting, add a Payment Line:
      Cash, a real amount less than the total (partial payment).
- [ ] Submit. Confirm the voucher shows an outstanding balance (visible via
      a Payment Voucher lookup, or the Payable dashboard stat moving).
- [ ] Try a Payment Line with method = Card or Benefit Pay — confirm a Bank
      picker appears and is required for those methods, but not for Cash.
- [ ] Confirm Credit does **not** appear as a usable option here (it's a
      sales-side-only concept, meaningless on an intake voucher).

### 4.7 Cancel

- [ ] **(Staff)**: confirm there's no Cancel option available on a submitted
      Purchase Voucher at all.
- [ ] **(Admin)**: cancel one of the test vouchers from §4.1–4.6 that has
      **not** had its phone sold. Should succeed cleanly, stock/phones
      reverted.
- [ ] Sell one of the test phones via POS first (§5.1), *then* try cancelling
      its Purchase Voucher as Admin — should be **blocked** with a clear
      message naming the Shop Sale that's in the way, not a raw error.
- [ ] **Clean up**: cancel every test voucher from this section once you've
      confirmed what you need to, in this order — cancel any Shop Sale that
      references a phone first, then cancel the Purchase Voucher, then
      delete both if you want a fully clean slate (Desk's document view has
      Delete once cancelled).

---

## 5. Point of Sale

All from the **Point of Sale** tile.

### 5.1 New phone sale, Inclusive VAT

- [ ] Scan/search for a tracked New-phone IMEI from §4.1, add to cart.
- [ ] Confirm the live VAT preview appears under the cart total (Net / VAT /
      Total, all 3-decimal), and updates as you edit the price.
- [ ] Pick a real Customer (use **Search by Phone** to find one, or create
      new inline).
- [ ] Complete sale with Cash. Confirm success banner, Print Receipt / Print
      Invoice / Void Sale buttons appear.
- [ ] Open the printed Standard invoice (via the print buttons, or
      `/printview` directly) — VAT amount is shown explicitly (New-phone
      invoices must show VAT).

### 5.2 New phone sale, Exclusive VAT

- [ ] Same as above but with a phone/line set to Exclusive. Confirm Total =
      price × 1.10, and the preview reflects that live.

### 5.3 Used phone sale (PMS)

- [ ] Sell a Used-phone IMEI from §4.2.
- [ ] Confirm the cart shows **no VAT breakdown at all** for this line (PMS
      rule: never show VAT amount) — no Net/VAT preview line for a
      Used-only cart, in either VAT mode.
- [ ] Complete the sale. Print the PMS invoice — confirm it explicitly does
      **not** state a VAT amount anywhere, just that VAT is included under
      the margin scheme.

### 5.4 Accessory sale

- [ ] Scan a real Item Barcode (or use Browse Accessories panel) to add an
      accessory to the cart.
- [ ] Complete the sale. Confirm `Item.current_stock` decreased by the
      quantity sold.

### 5.5 Mixed sale (New phone + accessory)

- [ ] One cart with a New phone and an accessory. Confirm this is **allowed**
      (New + accessory is fine — only Used mixing with anything standard-VAT
      is blocked).
- [ ] Try adding a **Used** phone to a cart that already has a New phone or
      accessory — should be **hard-blocked** with a clear message (not a
      warning this time — sales-side mixing is a hard rule).

### 5.6 Batch/untracked phone sale (IMEI captured at the till)

- [ ] Scan the Box UPC from §4.3. Confirm the cart line shows a "from batch"
      badge and an IMEI input (no pre-existing tracked unit).
- [ ] Enter a fresh IMEI at the till, complete the sale.
- [ ] Confirm a new tracked `Phone` record now exists for that IMEI, and the
      batch's remaining untracked quantity dropped by 1.
- [ ] Try scanning the same UPC again after exhausting all untracked units —
      should show a clean "no units left" message, not an error.

### 5.7 Payment methods

- [ ] Complete a sale with a **Mixed** payment: part Cash, part Card
      (pick a real Shop Bank), using "+ Add Payment" and the "remaining"
      auto-fill.
- [ ] Try completing a sale to a **Walk-in Customer** with Credit as the
      payment method — should be **blocked**.
- [ ] Try completing a sale of a phone (any phone) to a Walk-in Customer,
      with any payment method — should be blocked (phones require a named
      customer, accessory-only sales don't).
- [ ] Complete a real Credit sale to a named (non-walk-in) customer — should
      succeed, and the customer now has an outstanding balance (confirmed in
      §6).

### 5.8 Search & quick-select

- [ ] **Search by Phone** — type a customer's phone number, confirm it finds
      and fills the right customer.
- [ ] **Search Phone by Brand/Model** — confirm it finds only In Stock units,
      excludes already-sold ones.
- [ ] **Quick-select panel** (the 5 tiles) — confirm it shows real recently-
      sold products, and tapping one behaves correctly for each of the three
      cases: accessory (straight to cart), tracked phone model (opens the
      search dialog pre-filled), batch-only phone model (adds a from-batch
      line directly).
- [ ] **Browse Accessories** panel — confirm tapping an item adds it to cart
      the same way scanning its barcode would.

### 5.9 Void Sale

- [ ] Immediately after completing a sale (within the same session), click
      **Void Sale**. Confirm the phone returns to `In Stock` (or the batch's
      untracked count is restored, if it was a batch sale) and accessory
      stock is given back.
- [ ] **(Staff)**: confirm you can only void your *own* recent sale, and only
      within the staff void window — try voiding an old sale (or another
      staff member's) and confirm it's blocked.
- [ ] **(Admin)**: confirm Admin can void any sale, any age.

### 5.10 Blocking rules — quick negative-test sweep

- [ ] Try adding the same IMEI to the cart twice, or selling an already-sold
      IMEI — blocked with a clear message.
- [ ] Try selling more accessory quantity than `current_stock` — blocked.
- [ ] Confirm every block in this section shows a clean in-app message, never
      a raw stack trace or blank screen.

---

## 6. Customer Receipt

- [ ] From the Credit sale in §5.7: open **Customer Receipt**, pick that
      Shop Sale, confirm the outstanding balance shown matches what's owed.
- [ ] Enter a partial payment, submit. Confirm the balance reduces by exactly
      that amount (check again from a fresh Customer Receipt against the
      same sale).
- [ ] Pay off the rest with a second Customer Receipt — balance should reach
      zero.
- [ ] Try overpaying (more than the outstanding balance) — should be
      blocked.
- [ ] **(Staff)**: confirm Staff can submit a Customer Receipt but **cannot**
      cancel one.
- [ ] **(Admin)**: cancel a Customer Receipt — confirm the balance reopens
      by that amount.

---

## 7. Payment Voucher

- [ ] Against the partially-paid Purchase Voucher from §4.6: open **Payment
      Voucher**, pick that voucher, confirm the outstanding balance shown is
      correct (total minus what was already paid at intake).
- [ ] Submit a payment covering part of the remainder. Confirm the balance
      updates correctly on a fresh lookup.
- [ ] Try overpaying — blocked.
- [ ] **(Staff)**: cannot cancel a Payment Voucher. **(Admin)**: can, and the
      balance reopens correctly.

---

## 8. Shop Bank

- [ ] **(Admin)**: `+ Add` a new bank account (name + account number).
      First one created becomes the default automatically.
- [ ] Add a second bank, mark it default — confirm the first one's "default"
      flag clears automatically.
- [ ] Try disabling or deleting the current default bank — should be
      blocked with a clear message (must set a different one as default
      first).
- [ ] **(Staff)**: confirm the bank picker (Card/Benefit Pay rows in POS and
      Purchase Voucher) still works — Staff has read-only access, just no
      `+ Add`/edit.

---

## 9. Reports

Confirm every report below **loads with real data** for Admin, and that the
two Admin-only ones **hard-block Staff** with a clean permission message (not
a blank page or raw error). Where a report has filters, try at least one
(a date range, a brand/model text filter) and confirm it actually narrows the
results.

**All roles:**
- [ ] IMEI History Report
- [ ] Sales Report (has a chart — confirm it renders, not just the table)
- [ ] Purchase Report
- [ ] Accessory Purchase Report
- [ ] Customer Report
- [ ] Supplier Report
- [ ] Inventory Report
- [ ] Daybook
- [ ] Cash Book
- [ ] Bank Book

**Admin only:**
- [ ] Profit Report — loads for Admin, blocked for Staff.
- [ ] VAT Report — loads for Admin, blocked for Staff.

---

## 10. Registers cross-check

This is the one place worth actually doing the arithmetic by hand, since it's
the closest thing this app has to a financial statement.

- [ ] Pick a date range covering everything you did in §4–§7. In **Daybook**,
      confirm every cash/card/credit movement you made shows up, In and Out
      both directions.
- [ ] In **Cash Book**, confirm the running Balance column is mathematically
      consistent: each row's balance = previous row's balance ± that row's
      amount. Hand-check at least 2–3 rows.
- [ ] In **Bank Book**, view a single bank (should show only that account's
      movements) and then view with no bank filter (should combine every
      bank, and say so explicitly in the message above the table).

---

## 11. Receivable / Payable

- [ ] Dashboard's Total Receivable stat should match the sum of every
      customer's real outstanding Credit balance after §5.7/§6.
- [ ] Total Payable should match the sum of every supplier's real
      outstanding balance after §4.6/§7.
- [ ] After fully settling everything in §6 and §7, both stats should read
      back down to whatever they were before you started (or exactly zero,
      if you started from a clean database).

---

## 12. Print formats

For each, open via the print button on a real document and confirm it
renders correctly (no "None" text, correct brand/model, correct totals):

- [ ] Shop Sale Standard Invoice (shows VAT explicitly)
- [ ] Shop Sale PMS Invoice (never shows VAT)
- [ ] Shop Sale Standard Receipt (80mm — check it doesn't render A4-padded)
- [ ] Shop Sale PMS Receipt (80mm)
- [ ] Purchase Voucher Summary
- [ ] Self-Billed Purchase Voucher Invoice — renders Used lines only; if the
      voucher has no Used lines, confirm it shows an explanation instead of
      a blank signature line

---

## 13. Camera scanner (hardware)

This needs a real tablet or phone with a working camera — can't be verified
from a desktop browser.

- [ ] On the actual device that will sit at the counter, open POS, tap the
      camera scan button, confirm the camera opens and a real barcode/IMEI
      scan populates the field correctly.
- [ ] Same check inside Purchase Voucher's Capture IMEIs dialog.
- [ ] Confirm scanning works over whatever connection the till will actually
      use day-to-day (HTTPS, or a tunnel, or the same-network `mobileshop.local`
      hostname) — camera access is blocked by the browser on plain HTTP to
      anything other than `localhost`.

---

## 14. One real thermal print

Once the thermal printer is connected:

- [ ] Print one real Standard Receipt and one real PMS Receipt from an actual
      till sale. Confirm the 80mm sizing looks right on real paper (not just
      correct on screen) — this is the one thing that genuinely can't be
      checked without the hardware in hand.

---

## Notes for anything you find

For each issue you hit, note down: which numbered step, what you expected,
what actually happened, and whether it's reproducible (try it again once
before assuming it's a fluke — some of the real bugs found during earlier
development only showed up on a second attempt).
