# MuSo Admin Portal — Session Context / Change Log

**File worked on:** `muso-admin-portal.html` (single self-contained HTML wireframe — inline `<style>` + one big inline `<script>`; render-driven: state vars + `render()` rebuilding `#pane`/`#rail`; display-only, nothing persists).

**How to verify changes:** extract the largest `<script>` → `node --check`; then a DOM-stub harness dispatching synthetic delegated `click`/`input`/`change` events and asserting on `#pane` innerHTML; plus a full render sweep over every master + dashboard + membership + POS. All changes below were verified this way (syntax clean, sweeps clean).

**Nav / masters:** left rail driven by `ORDER` + `PHASE` + `GROUPS` + `M` registry; a standalone **Dashboard** item sits at the top (outside `ORDER`, so the 14 masters keep steps 01–14). Group 6 (`Membership`) was appended (`tiers`, `members` = steps 15–16).

---

## Changes made this session (chronological themes)

### 1. Voucher / series discounts (Discounts module)
- New **Code type** toggle on the discount form (Discount-code method): **Single code** vs **Code series**.
- Series = **prefix + From/To**, auto-padded to the width of "To" (`1–100 → JSW001…JSW100`), live preview, validation (blank prefix, From>To, cap 1000).
- Data: `d.codeType` (`"single"|"series"`), `d.series = {prefix, from, to}`. Helpers: `seriesPad/seriesCount/seriesCode/seriesCodes/seriesLabel/isSeries/codeLabel/seriesError` (near `genCode`).
- `codeLabel(d)` used everywhere the code shows (listing, detail header, summary panel) so a series reads `JSW001–JSW100`.
- Detail **Code & link** tab lists all generated codes (capped 1000) with Unused/Redeemed pills + a **Download all codes (CSV)** stub.
- Seed example: **JSW001–JSW100**, ₹500 off, **staff only** (`staffOnly:true`).

### 2. Dashboard (new, default landing)
- `VIEWS.dashboard()` + `M.dashboard` (standalone top rail item); initial `current="dashboard"`; `head()` special-cases it (hides New/History, shows a "Today" chip).
- **KPI tiles with inline-SVG icon badges** (`.kpi-ico`, colored variants). Two rows of tiles:
  - Orders today / Revenue today / Capacity in use (real) / Active discounts (real).
  - Adults today / Children today / Repeat visitors / Schools booked (sample).
- Panels: Today's sessions (real slots + availability pills), Needs attention (unapproved discounts, draft programs, tight slots, upcoming blackouts — each `data-goto` to its master), Revenue last-7-days CSS bars, Capacity-by-product meters, Booking sources split bar, Today-vs-best-day comparison, Recent orders, and a **Membership** row (by-tier breakdown + "needing attention" list).
- Time-based figures are clearly tagged **sample** (via `DASH` constant); structural ones (capacity, counts, orders) are real.
- New CSS: `.bars/.bar/.bar-fill`, `.meter/.meter-fill`, `.splitbar`, `.cmp*`, `.kpi-*`, `.attn*`, `.dash-row`.

### 3. Membership (new — Group 6: `tiers` + `members`)
- **Membership tiers** master: configurable plan **name**, **visits** (a number or **Unlimited**), **validity** (3mo/6mo/Yearly), **price**, and **per-plan discounts** (Birthday / Events / Workshops, editable). Seed: Silver/Gold/Platinum (Platinum = Unlimited). Dedicated `tierForm()` + `tierDetail()`.
- **Members** master (directory): KPI tiles + table keyed on the **registered adult-child pair**. Detail with tabs **Summary / Purchase history / Redemptions & visits**, plus actions **Renew** and **Record redemption (OTP step)**.
- **Model rules (client-confirmed):** 1 visit = 1 adult + 1 child; the registered adult+child pair captured at purchase is the only pair that may redeem; Platinum unlimited, Gold/Silver limited.
- Member create/edit form (`memberForm()`): Plan, registered **adult** and **child** (each with optional **Date of birth** + an **Age** field that auto-fills from DOB; Age **required for child**, optional for adult), Phone (tel), Email. Reuses `ageFromDOB/ageLabel` + the `[data-dob]→#age_*` input handler.
- Seeds: `MEMBERSHIP_TIERS`, `MEMBER_TODAY`, `MEMBERS` (5 members incl. Aarav/Gold, an Expiring-soon, an Expired). Helpers: `tierOf/isUnlimited/visitsBalance/visitsLabel/pairLabel/memberStatus/memberStatusPill/tierPill`. `MEMBER_SUBS`, state `selMember/selTier/msub/redeemStep/redeemVisits`.
- **Members searchable by mobile number** (also name/ID): each row carries a `data-search` haystack; a `[data-member-search]` input handler filters rows live (no re-render, keeps focus), updates an "N of M members" count, shows a no-match row.
- Tier detail Terms rendered as a **bulleted list**.
- Dashboards/labels: tier pills `.tier-silver/.tier-gold/.tier-platinum`; fixed the member-detail summary key/value table padding (was flush to card edge).
- "Approval status" field renamed to **Status**; "Total times it can be used" → **"Total times it can be used across users"** (earlier discount copy tweaks).

### 4. Products & Programs (earlier in session)
- Products & add-ons: **Capacity always visible** (Optional when Building Pool on, Required off); row actions **View / Edit / Activate-Deactivate switch**; Building Pool **off by default**.
- Programs: added an **Inventory** section (shared `inventoryFields()` helper); **Booking rules** and **Tax** split into separate sections; removed "Default capacity per batch"; added a **View** button.
- Program (camp) detail Register & running days: added a **Batch column** (all batches, combined seat count); removed the "Who is told" card + explanatory notes; removed the "Add day" button.
- Availability standardized to **Sold out (0%) / Fast filling (≤25%) / Available** via a shared `availState(left,cap)` helper.
- Removed the standalone **Events** and **Camps & workshops** nav pages (consolidated under Programs); adding a batch **auto-generates its daily sessions** (`parseDMY`/`genBatchDays`).

### 5. Orders / POS flow
- **Customer step:** Phone + Customer name made **mandatory**; new mandatory **Location** field = the **customer's own city/area** (free-text `bookCustLocation` → `bookingCustomer.location`); Next is gated until Phone/Name/Location are filled (`.fld-miss` red state, `bookingStep1Tried`).
- **Location auto-fill + seeds:** added `location` to the 3 `BOOKINGS` seeds and carried it through `CUSTOMERS`; picking an existing customer (phone lookup) and edit-load both auto-fill Location.
- **Payment methods:** Cash / Card / UPI / Voucher (default Cash).
- **Attendee Age** auto-calculated from DOB (editable); **Name & Age mandatory for child** attendees.
- **Child validation:** under adult-required child categories (Child 3–11, Baby), shows "Purchase of an adult ticket is mandatory for this age group" — muted normally, red warning when a child is in the cart with no Adult (`adultQty` now sums across multi-slot).
- **Membership redemption reworked to visit-based (1 visit = 1 Adult + 1 Child):** the old "Redeem from your membership" 4-category free-mix steppers (which wrongly deducted a visit *per ticket*) were replaced by a **full-width horizontal membership box** with **two checkboxes** — (1) "Use a membership visit for this booking" (admits 1 child free, disabled at 0 remaining) and (2) "Add the accompanying adult to this visit, free" (enabled only after 1). Booking-level flags `bookingRedeemVisit`/`bookingRedeemAdult`; **only 1 visit per booking/day**. Covered people are ₹0 free heads (1 Child + optional Adult), a green "Membership visit · 1 Adult + 1 Child" ₹0 summary row shows, `adultQty` counts the redeemed adult, and "Pay for tickets" is now full-width for extras. `globalPassesRemaining()` and the used/remaining counter now count **one** visit, never per-ticket. Removed the `redeemStep`/`data-book-redeem`/`.redeem-*`/`.ticket-cols` code.
- **Visit is MuSo Tour only, and only for a plain Tour booking:** the box (and all its counting/free-heads/summary) shows **only** when the **MuSo Tour** (`P` with `c:"MUSO"`) is in the cart **with no add-ons and no other products** — adding any add-on (qty>0) or other product **hides** it and stops the redemption counting. Programs are allowed alongside (they get the member program discount, not a visit). Gated via `membershipVisitApplies()` (used by both `bookingForm()`'s `hasTour`/effective `redeemVisit`/`redeemAdult` and `globalPassesRemaining()`). The box renders **between the product cards and the Tour detail** (`item-cards` → `membershipBox` → `productDetails`).
- **Members get an automatic discount on programs:** the pre-existing `memberOnly` auto-discount branch (member + any program → 10% of `progTotal`) is wired; its seed was renamed **"Members — 10% off programs"** (was "…all Workshops"; `cond.prods:["Program"]`, Approved) so it reads/works for any program type (workshop, camp, event), shown as a green `auto` summary row.
- **Check-in/out + manual discounts were built then REVERTED** at the user's request (only the multi-code discount work was also reverted back to a single code box).

### 6. Booking combinations (multiple date-time + tickets per item) — supersedes the old multi-slot model
- **Every cart entry is now `{combos:[…]}`** — an array of independent combinations. Product/add-on combo = `{date, slot, qty, redeem}` (add-on has no `redeem`); program/camp combo = `{batch, qty}`. This replaced the earlier one-date-plus-multi-slot-set shape (`{date, slots:{…}}`) and the flat program `{batch, qty}`.
- **Slot picker is back to RADIO** (one slot per combination); the multi-select checkbox ("tap to select one or more" / N-selected) is gone. `slotBar`→**`comboPicker(pfx,list,combo)`** renders a per-combo calendar (month tracked on `combo.calMonth`) + radio time-slots.
- **UI = stacked "Ticket N" blocks** (`.combo-block`), each with the same elements (calendar + radio slots + ticket steppers) + a per-row **Remove**; an **"Add Ticket"** button (`data-book-add-combo`) appends another combination. Applies to products (MuSo Tour), add-ons (Mini Golf, Lost Code of Play) and camps/programs (Kinetic Kids — a "combination" = a batch, so multiple batches per order).
- Helpers rewritten to iterate `combos`: `combosOf`, `slotFor`, `batchFor`, `priceIn`, `qtySum`, `catQtys`, `adultInCfg`, `prodLine/addonLine/progLine`, `productRedemption` (key `i|comboIdx|cat`), `collectCombos` (attendee heads, labeled by date+slot/batch), `cartRows` (one group per combo), `hasBookingItems`, `membershipVisitApplies`. New shared `firstComboFor(kind,id)`.
- Handlers: qty prefix now `kind|id|comboIdx|cat|delta`; `data-book-slot` = `kind|id|ci|slotName` (radio replace); `data-book-cal-date`/`-cal-nav` carry `ci` (per-combo month); `data-book-batch` = `prog|key|ci|bi`; new `data-book-add-combo` / `data-book-remove-combo` (removing the last combo deselects the item). Removed the dead `data-book-quick`, `slotOf`/`addonSlotOf`, `progBatchOf`, `sumSlotted`/`sumFlat`, and `.cal-slots-count` usage.
- Wireframe note: each combination renders its own calendar (heavy but clear); the membership visit stays booking-level (one visit even if the Tour is booked across multiple date combos).

### 7. POS discount application (latest)
- **Any code now applies** (not just IND50). The typed-code matcher was broadened to match Approved **code-based** discounts by `d.n` (MUMKIDS150), `cond[].code` (IND50), or **series voucher range** (JSW001–JSW100). Updated in `bookingForm()` math and the twin `recalcBookingSummary()`.
- **Auto-applied discounts** now evaluate against the cart (simplified, Approved only): **Membership** (member linked → 10% subtotal), **Add-on bundling** (product + add-on → 10% add-ons, once), **Quantity break** (≥ tier qty of one category → tier %), **Members-workshops** (member + program → 10% program). Pending "Prebook" never applies.
- **Everything stacks** (user chose stack-all, no biggest-wins): each auto discount + the typed code render as their own green summary rows (autos tagged `auto`), plus the existing member add-on line; grand total floors at 0. New math: `catQtys/anyQty/hasProduct/hasAddon/hasProgram/maxCatQty/pctOf`, `autoDiscs[]`, `codeDiscAmt`, `discAmt=min(subtotal, code+auto)`.

### 9. Shop / Merchandise module (new — Group 7)
- New **Group 7 · Shop** with four masters (ORDER now 21 masters): **Shop products** (`shopitems`), **Shop sales** (`shopsales`), **Shop coupons** (`shopcoupons`), **Shop reports** (`shopreports`). Modeled on the Zoho "MUSO Shop Inventory" app + Scope §5 + the `E- Invoice Format.pdf`.
- **Reuses the existing Tax-rate & HSN masters**: added the missing goods GST rates **5% / 12% / 28%** (CGST+SGST+IGST component rows) to `TAXRATES` and **8-digit goods HSN** rows to `HSNCODES`. A Shop product's `tax` array references `TAXRATES` codes; GST% is derived (`shopGstPct`), never hardcoded.
- **Shop products** (`SHOP_ITEMS`): PD code, barcode, name, category, base price, GST (via tax master), MRP, HSN, stock. KPI+search listing, `createForm(SCHEMAS.shopitems)`, `shopItemDetail`, and an **Add stock** drawer (`shopAddStockCard` → increments `stock`).
- **Shop sale** (single-page `shopSaleForm`): customer phone (**required**) + name/email + **optional customer GSTIN**; item table (PD select → auto name/avail/base/GST/MRP, barcode-scan stub, qty steppers, add/remove rows); Sub Total; **Discount Type** (Employee 15% / Gift 100% presets) + on-the-spot % + **approved-only coupon** + membership; **per-line GST** (CGST/SGST split, rate from each item's tax master) via `shopSaleCompute`; Grand Total + **amount in words** (`amountInWords`); payment (Cash/POS-UPI/POS-Card/None) + split; Save/Print/Send stubs. Save decrements stock and opens the invoice.
- **Invoice** (`shopSaleDetail`, `DETAIL.shopsales`): GST **Tax Invoice / e-Invoice** per the PDF — seller `MUSO_SELLER` (GSTIN 27AAQCM9400K1ZZ), buyer bill-to, particulars w/ HSN·qty·rate·amount + CGST/SGST rows, total, amount-in-words, HSN/SAC summary table, bank details, authorised signatory, "Computer Generated Invoice". IRN/Ack/QR block shows **only when the buyer GSTIN is present** (e-invoice).
- **Shop coupons** (`SHOP_COUPONS`): approval-gated — only `status:"Approved"` codes apply; form carries **Prepared/Proposed/Approved by**; listing shows approval pills. **Separate** from the main Discounts module (per the choice).
- **Shop reports** (`shopreports`): sales summary (revenue / GST collected / net / by payment mode) + **HSN/SAC-wise quantity + CGST/SGST breakup** (mandatory for the GST return).
- Handlers: `data-shop-*` (pick/qty/addrow/removerow/disctype/coupon-apply/pay/split/scan/save/print/addstock) in the click/change/input listeners; `data-open="shopitem|shopsale|shopcoupon:idx"`. New CSS: `.shop-seg`, `.shop-sale-table`, `.invoice`/`.inv-*`. Verified: node --check + 22/22 shop harness + 23-view sweep (0 failures).
- **Deferred / stubs**: Credit notes (not selected); barcode scanner, Print, SMS/WhatsApp, Tally, e-invoice IRN generation are display stubs; single flat intra-state CGST/SGST split (no IGST branch).

### 8. Reception Check-in / out (QR scanner page)
- **Check-in / out** master under the **Calendar** group (in `ORDER`/`PHASE`/`PREREQ` after `bookings`; `M.checkin`, `head()` special-case hides New/History, `VIEWS.checkin()`). Not a CRUD master — a single reception/kiosk view.
- Simulates a QR scanner: an animated scanner viewport (`.scanner`/`.scanline`, `prefers-reduced-motion`-guarded) + a list of **today's expected bookings** as tappable QR-thumbnail cards (`fauxQR()` = decorative deterministic SVG, not a real QR). Tapping a card = a "scan".
- Flow driven by `checkinRef` (ref on scanner, `null`=waiting) + `CHECKINS` (`{ref:{inTime,outTime}}`, in-memory): **Waiting → Scanned (ticket details + Check in) → Checked in (Check out) → Checked out (Scan next)**. Re-scanning a ticket reflects its saved state.
- Ticket details read straight from the `BOOKINGS` seed (visitor, product, date, slot, tickets, per-attendee list, total). **Online bookings are always prepaid** — no payment gating; payment shows only as a neutral "Paid · <mode>" line.
- Handlers: `data-scan` / `data-scan-reset` / `data-checkin` / `data-checkout`; helper `nowTime()` (real `Date`, fine in page runtime). Self-contained CSS block (`.scan-*`, `.ci-*`, `.qr`, `.pill-warn/off`, `.btn-out/lg`). Verified: node --check + DOM-stub flow (10/10) + full 18-view sweep (0 failures).

### 10. Iterative refinements (this session)
Many small, client-driven edits after the Shop build. Grouped by area. All verified with `node --check` + DOM-stub harness + full render sweep (0 failures); one change also live-inspected in Chrome (chrome-devtools MCP).

**Foundations — Tax rates**
- Trimmed the Tax-rate form: removed **"Which tax this is"** (`component`) and **"In force from/to"** (`effective_from/to`) and the whole "When it applies" condition block. Rate is **required + numeric only**; **"In use" → "Live"**. `createForm` "tax rate" branch (Basics / Rate & validity).

**Foundations — HSN / SAC codes**
- Tax is now **multi-select** ("Tax rates", `tax_rate_ids[]`): each `HSNCODES` row carries a `tax` array (CGST+SGST+IGST); the listing shows **total % + CGST/SGST breakup**. Helpers `hsnTaxCodes / hsnComponents / hsnTotalPct / hsnHalfPct / hsnCompLabel`.
- **"Goods or services" defaults to HSN**; **Unit removed** (moved to the product); "Description on the invoice" → **Description**.

**Discount type wording (both modules)**
- **Discounts & offers** and **Shop coupons** discount type is now **Percentage / Flat amount** (capitalised). Shop-coupon data model migrated: `type` `"%"/"₹"` → `"Percentage"/"Flat amount"`, and every consumer (`shopSaleCompute`, listing, detail, CSV, preset resolver) updated.

**Shop categories (new master)**
- `SHOP_CATEGORIES` master under Foundations (`shopcats`; schema Category name + 2-letter Code), replacing the hardcoded `SHOP_CAT_PREFIX`. `shopPdFor(cat)` derives the PD prefix from it; a listing shows per-category product counts.

**Shop products — pricing & IDs**
- Custom `shopItemForm()`: removed manual Base & Sale-GST inputs. **HSN drives GST** (shown **read-only** as a label with CGST/SGST breakup); **Base auto-calculated** = Selling − GST (GST-inclusive), read-only; MRP → **Selling price**; added **Cost price** (defaults to Base, editable). Flow is **Tax rate → HSN → Product** (e.g. Selling 1200, GST 20% → Base 1000).
- **Product ID (PD)** auto-generated from Category on create — the manual PD field is **removed**; PD still shown on the product detail. `data-shop-item-save` now **persists** a new `SHOP_ITEM` (was a no-op) and opens its Barcodes tab.

**Shop products — per-unit barcodes**
- Removed the single product `barcode`. Each product carries **`barcodes[]` — one unique 13-digit code per unit** (`shopUnitBarcode` = `"89"` + 5-digit PD + 6-digit seq); `ensureShopBarcodes(it)` tops up to `stock` on seed load, create, and **Add stock**. `shopItemByBarcode` searches the arrays.
- Product detail is **tabbed: Details | Barcodes**. Barcodes tab = table of every unit (# · Barcode · **Print** per row + **Print all**, wireframe stubs). Inventory listing/CSV show a barcode **count**; the sale form's Barcode column was removed (scan still resolves a unit code → product).
- **Actions box** (Add stock / Edit product) moved out of the tab body into a **persistent sticky right column** (`.disc-summary-col` inside `.two`) so it survives tab switches — fixed a real bug where unbalanced `<div>`s had nested it inside a card.

**Shop sales**
- Item details: added **HSN/SAC** column; Base & Selling computed live (`shopBaseOf` / `shopSellingOf`) — fixed a stale hardcoded base. **Customer GSTIN** input added.
- **Discount redesign — two mechanisms**: (1) **Coupon code** — typed, or **Employee/Gift preset pills** that *fill* the field (Apply then applies); presets resolve via `shopCoupon()` (`SHOP_PRESETS`, kept out of the coupons master). (2) **Instant discount** — custom % → **authorised manager** dropdown (`SHOP_MANAGERS`) → **Send OTP** (fixed demo code shown) → enter matching code to apply, with a **Discount description**; Remove clears it. Removed the old Discount-type segment & On-the-spot %. Both stack additively (cap 100%). An applied coupon has a **Remove** link.
- **Totals model**: **Total (Selling) → Discount (off Selling) → Sub total → CGST / SGST breakup → Grand total** (`shopSaleCompute` returns `total/subtotal/factor/discAmt` + per-line `taxable`; seeds migrated `discType`/`spot` → `coupon`/`instantPct`/`instantMgr`/`instantDesc`; added a seed sale with an instant discount).
- Listing: added a **Discount** column; the row action is a single **View details** button opening a **tabbed detail: Details | Tax invoice** (`ssub`). Details shows customer email, coupon, instant %, **Discount description**, authoriser, and **Discount type / percentage tied to the coupon** (dash when no coupon). The invoice discount row shows the total % only. **Split payment removed**; **Shop reports hidden** (dropped from `ORDER`).

**Discounts & offers**
- Removed **Created by / Approved by** from the form; Status = Draft / Pending approval (capitalised). Right-hand **Approval box** (Created by · Approved by · Status pill + full-width **Approve** button when Pending) — persistent **sticky** across all detail tabs.

**Shop coupons**
- **Approve** button on the coupon detail (Pending → Approved), hidden for Draft; removed Approved from the Status dropdown; removed Prepared/Approved by from the form.

**Membership & booking**
- **Membership tier form**: added **Striked price**; the **Products & time slots** section is now a flyout (searchable product multi-select + per-product slot chips), and is **required** with save-time validation.
- **Membership visit** moved **per-combo** (per ticket), sitting between Available Slots & Pay for tickets, and **gated to the MuSo Tour 14:00–19:00 slot only**. When applied with no paid tickets the cart shows **1 Adult + 1 Child (free)** and a **Take payment · ₹0** CTA.
- Member & New-order forms: added **Pincode**; Location & PIN are both **Optional** but at least one is required (frontend validation). POS gained a **customer GSTIN** field.

**Dashboard**
- "In the building now" changed from occupancy cards to a **table** — Product · Date · Start · End · **Booked** (Adult/Child) · **Checked-in** (Adult/Child) — sourced from `DASH.building`. (Emergency-headcount note and the Booked-from column later removed.)

**Visitor categories**
- Added an **"Is adult"** switch to bifurcate adults from children in the live headcount.

### 11. Careers + Departments merge, capacity relocation, careers refinements (latest session)
Verified throughout with `node --check` + a DOM-stub render harness + full render sweep (0 failures); selected changes live-checked in Chrome (chrome-devtools MCP).

**Merge — Careers & Departments (People group)**
- 3-way `git merge-file` folded `~/Downloads/muso-admin-portal_5.html` in (only 3 conflicts, all resolved). Added **Group 8 · People** with two masters: **Departments** (`departments`, step near progcats) and **Careers / roles** (`career`, appended). `affiliates`/`paymodes` stay removed from `ORDER`.
- Careers: `careerDetail`, `careerCreate`/edit (`careerEditValues`, `nextRoleId`), `appTable`, role states (`roleState`/`rolePill`/`ROLE_STATES`), day-countdown (`careerDaysLeft`), CV/JD PDF stubs, live search (`careerSearchRows`/`syncCareerSearch`), sortable tables (`sortRows`/`sortTh`). Seeds: `CAREER_OPENINGS`, `CAREER_APPLICATIONS`, `CAREER_QUERIES`, `APP_STATUSES`, `HR_EMAIL`. Departments: `DEPARTMENTS`, `deptEditLayer`, ratio helpers.

**"Active" label standardisation**
- Renamed 5 strong on/off-switch candidates to **Active** (via perl): `Switched on`, `Offered`, `Available to book` (×2), and the careers `Listing` toggle → **Active** (keys unchanged: `is_active`/`active`).

**Capacity moved to the time-slot level**
- Removed product/add-on-level Capacity input from `inventoryFields()` (kept the Building-Pool toggle) and program-level capacity from `PROGRAM_SCHEMA`/`CAMP_SCHEMA`/`EVENT_SCHEMA`/`PROG_SINGLE`/`PROG_MULTI`. **Batch capacity stays** (the program's schedule unit).
- Added a **Capacity** field (`id="slotCap"`) to the time-slot flyout (`slotFormCard`), wired into add/edit save. Availability math already keyed off `slot.cap` (`slotLeft`/`availState`), so this surfaced an existing field.
- **Reserved capacity removed** from the slot form (dropped `id="slotHold"` + its save wiring; add now uses `hold:0`).

**Capacity blocks (Products & add-ons detail)**
- Reason dropdown gained **School Booking** & **Birthday Booking** (schema enum + `blockFormCard` select).
- Added an **Edit** button per block (`data-edit-block`, `editBlock` state, prefill + update-in-place vs push, `availability()` table gained the action column, colspans 6→7). Seeded 3 dummy blocks on MuSo Tour (School Booking / Birthday Booking / Private group).

**Careers detail page**
- **Status column hidden** from the Applications box (`appTable` header + per-row select removed; colspans 8→7).
- **Brought in line with the platform detail pattern**: tidied the `.detailhead` to one clean line, and split the body into **`.subs` sub-tabs — Details | Applications · N** (state `carDsub`, handler `data-cardsub`, reset to Details on open). Content is **full-width** — a sticky "At a glance" + "Actions" aside was built (mirroring shopItemDetail) then **removed at the user's request**; only the sub-tabs + tidy header remain.

### 12. School Bookings module (Scope §8) — new "Bookings" group (latest session)
Verified with `node --check` + DOM-stub render sweep (0 failures across all 25 masters) + logic tests (create/edit persist, payment flips balance to ₹0, proforma math ₹87,000 + 18% GST = ₹1,02,660, valid `%PDF`) + live chrome-devtools checks (list + detail/proforma render consistently with the Shop invoice).

**Nav**
- New group **`9 · Bookings`** (`GROUPS`, colour `#1E7F3C`) with one master **`school`** ("School", `PHASE` idx 8). Three supporting masters added under **Foundations** (group 0): **`fboptions`** (F&B options — adult/child pricing), **`schoolslots`** (school time slots + museum-tour mapping), **`entryprices`** (entry price presets). All wired through `ORDER`/`PHASE`/`PREREQ`/`M`/`SING`/`OPTIONS`.
- **Sponsored School (§9) intentionally NOT built** — only status/record seams remain.

**Model & masters** (seeds live in-memory like Careers; reload resets)
- `SCHOOL_BOOKINGS` — one merged record type (enquiry + booking) keyed by a **status lifecycle**: `Open → In Discussion → Confirmed → Live → Ended`, plus `Lost`/`Rejected` (`SCHOOL_STATUSES`, `schoolStatusPill`/`SCHOOL_PILL`). Fields cover enquiry (school/location/boards[]/grade/POC/email/phone/interested[]/prefDate/expected counts/note) + backend (visits[] each date+slot+headcounts, entry prices, food{students/adults/comp/optionIds}, optionals[], billing, comments, payments[], timeline[]). Seeded 5 records across statuses.
- `FB_OPTIONS`, `SCHOOL_SLOTS` (each maps to a MuSo-Tour retail slot for inventory), `ENTRY_PRICES`. Boards/interested are fixed `enum[]` chips (no master). Helpers: `schoolById`, `nextSchoolId`, `schoolCompute` (mirrors `shopSaleCompute`; 18% GST, complementary heads ₹0, `eInv` when GSTIN present), `schoolLog`, `schoolMapNote`, `schoolExportRows`/`SCHOOL_EXPORT_NAMES`.

**Views / forms / detail**
- `VIEWS.school()` — list with sub-tabs **Bookings & enquiries** (KPIs + status filter + live search + Download CSV + `data-open="school:i"` rows with per-row **View** + **Edit** buttons — Edit (`data-school-edit-row="<id>"`, intercepted before the row's View handler) opens the edit form for that record directly — + a single **`+ New booking`**) and **Reports** (`schoolReports()`: date-range toggle, KPI tiles, by-board + repeat-visitor tables, CSV). Simple list views for the 3 masters.
- `schoolForm(editing)` — one custom form driven by `schoolDraft` (deep-copied on open), always full. Its **enquiry-capture sections mirror the public School Booking form** (flow + headings + placeholders): **School Information · Education Board · Visit Details · Point of Contact · Interested In · Additional Information**; then the admin-only **Entry pricing** (entry-preset picker + per-head prices) · **Visits** · **Billing** · **Status & comments**. **Visits are their own full-width section**: each visit is a wide card laid out in multi-column rows (Date/Slot, then headcounts 4-up, then Food & F&B 3-up + option chips, then Optionals) so it stays short. **Food & F&B and Optionals (venue blocks) live inside each visit** (`visit.food` + `visit.optionals`), so every visit has its own — `schoolCompute` aggregates food across visits; the detail's Visits table shows per-visit Food and Optionals columns. Visit 1 is always present and can't be removed (`blankVisit()`); further visits add below via **+ Add visit**. New records start `status: Open`. (The separate lightweight "enquiry" form was removed — the booking form is its superset.) Education Board and Interested In are 6/2-chip multi-selects (SSC/ICSE/CBSE/IGCSE/IB/CIE; Museum Tour/Workshop) each with an always-shown **Other** free-text field (`boardsOther`/`interestedOther`), shown on the detail as `Other: …` tags. The **School Name** field is a **typeahead** (`data-school-typeahead` + `.ta-menu`/`.ta-item` CSS, sourced from `schoolDirectory()` = distinct schools already on file): typing shows matching schools in an in-place dropdown (no re-render, keeps caret), and picking one (`data-school-suggest`) auto-fills the contact block (location, boards, grade, POC, designation, email, phone, interested). `schoolCaptureForm()` reads DOM → draft (so add/remove-visit/preset preserve typed values); `schoolCommit(notify)` validates + persists; an **edit** save opens a "Save changes?" dialog (`schoolSaveConfirm`) with an "Email the POC" checkbox (default on) so any save can skip its email (unticked → logs "No email sent."). The same "Create booking?" dialog now also gates the create confirmation/acknowledgement email.
- `schoolDetail()` — `.detailhead` + `.two` (sub-tabs **Overview · Payments · Proforma invoice · Timeline** + sticky Actions aside). Payments: add offline/record online → `schoolAdvanceOnPayment` sets status automatically — any receipt (partial) → **Confirmed**, clearing the balance → **Live**. Once a payment exists (`schoolStatusLocked`) the status is **locked** — the edit form's status field becomes a read-only pill (unlocked, the form dropdown offers only Open/In Discussion/Lost/Rejected via `SCHOOL_MANUAL_STATUSES`; Confirmed/Live/Ended are automatic). **The Actions aside has no status dropdown** (removed) — status is shown read-only in the "At a glance" card and changed (when unlocked) via **Edit details**. Proforma: `schoolProformaInvoice(rec, mode)` renders the proforma (default) or the **e-invoice** (`mode:"einvoice"` — Tax Invoice · e-Invoice with QR + IRN/Ack); reuses `.invoice`/`inv-*` + `MUSO_SELLER` + `amountInWords`; **Download proforma/e-invoice (PDF)** via `schoolProformaPdf(rec, mode)` → `careerOpenPdf`; Upload-proforma stub. **When a GSTIN is on file** the Proforma tab shows a **Generate E-invoice** button (`data-school-einvoice-gen` opens an "e-invoice can be generated only once" confirm (`schoolEInvConfirm`); `data-school-einvoice-confirm` stamps `rec.einvoice={generated,on}`, logs a viewable email, opens the modal — guarded against re-generation); once generated it becomes **View E-invoice** (`data-school-einvoice-view` → `schoolEInvView` modal); no GSTIN → a note that only a tax invoice is raised on completion. Timeline: `.attn`-style feed, **sorted newest-first by date** (stable). **Email events (Shopify-style)**: website-enquiry acknowledgement (`ack`), send-email, status-change (incl. status changed via edit), edit and backend-create log via `schoolLogEmail`, which attaches a composed email snapshot (`schoolComposeEmail(rec,kind)` → {to, subject, body, kind, sentOn}; `ack` shows enquiry details, others a booking summary) to the timeline entry; those rows show a **View email** button (`data-school-view-email=<timeline idx>`) opening a modal (`schoolEmailView` state) with subject + ✓ Delivered badge + MuSo-branded body + Cancel/Resend (`data-school-email-close`/`-resend`). NB: rendered HTML passes through `scrub()` (line ~5353), which strips wireframe placeholder phrases globally — **avoid "to confirm"/"provisional"/"not stated"/"§…" in any user-facing copy** or it silently vanishes (the ack email used "to confirm" → reworded to "to finalise"). `schoolBackfillEmails()` runs once at startup to attach viewable snapshots to seed timeline entries that represent outbound mail (enquiry-received → `ack`; status/emailed/proforma → status/booking); inbound events (check-in, payment recorded) stay plain. Aside actions: Edit details, Send email (confirms "Email sent" then the timeline carries View email), Download proforma, Mark completed (→ Ended + e-invoice/tax-invoice note; shown once payment-locked).
- Handlers: `data-school-*` (new/edit/cancel/save, add/del-visit, add/del-optional, addpay/onlinepay, sendmail, complete, proforma-pdf/upload, status/filter/preset, range, msg-dismiss) + `data-schtab`/`data-schsub` + `school` clause in the `data-open` handler; `syncSchoolSearch()` in the render tail. `schoolDraft` reset on nav / New / list transitions.

### 13. Sponsored School module (Scope §9) — built on the School module (latest session)
Verified: `node --check` + render sweep (0 failures across **28** masters) + logic tests (create/edit persist, mark-completed → Ended + sponsor e-invoice raised, invoice buyer = sponsor, student-CSV upload derives girl/boy split, valid `%PDF`) + live chrome-devtools checks.
- **Sponsor-funded, not school-paid.** Same `9 · Bookings` group gains master **`sponsored`** ("Sponsored School"). Two supporting **Foundations** masters added: **`sponsors`** (JSW Foundation / Tata Trusts / HDFC Parivartan — name, GSTIN, state, address, email) and **`districts`**. Reuses F&B options / School time slots / Entry price presets.
- **Reuses the School engine**: `schoolCompute` (totals), `schoolProformaLines` (invoice lines), `SCHOOL_STATUSES`/`SCHOOL_MANUAL_STATUSES`/`schoolStatusPill`, `blankVisit`, `schoolMapNote`, `fbByName`, `MUSO_SELLER`/`amountInWords`/`.invoice`+`inv-*` markup, and the `pdf*` stack. Distinct **`data-sp*`/`data-spf*`/`data-spv*`** attributes + `spDraft`/`selSp`/`spsub`/`sptab` state keep it separate from School handlers.
- **Model** `SPONSORED_BOOKINGS` (id `SPS-###`) = School record shape **plus** `orgType` (Funded (NGO)/School), `sponsor`, `district`, `transport{need,vendor}`, `ngoGstin`/`ngoPan`/`authNumber`, `speciallyAbled`/`underprivileged`, `proofDoc`, `postVisit{remark,images[]}`, `students{uploaded,total,girls,boys}`, `feedback{sent}`, `invoice{raised,on}` — **no payments** (sponsor is billed). Enum sets: `SP_ORG_TYPES`, `SP_TRANSPORT`, `SP_INTEREST` (adds Introduction to AI session / Future Maker Program), `SP_BOARDS` (adds State board / Home schooling). Seeded 4 records across statuses.
- **Views**: `VIEWS.sponsored()` (Bookings & enquiries table with sponsor/district + Invoice status column, per-row View/Edit, `+ New booking` | Reports tab → `sponsoredReports()` by sponsor + district); simple `sponsors()`/`districts()` masters. **Form** `spForm` sections: Organisation (name/type/district) · Education Board · Visit Details (grade/date/expected/**transport**+vendor/proof-of-ID) · Point of Contact · Interested In · **Sponsor & funding** (sponsor/auth no./NGO GST+PAN) · Entry pricing · **Visits** (same per-visit food/optionals as School) · Impact & remarks (specially-abled + underprivileged) · Status & comments. **Detail** `spDetail` tabs: Overview · **Sponsor invoice** (`spInvoice`/`spInvoicePdf` — Tax/e-Invoice **billed to the sponsor**, beneficiary = school, raised on completion) · **After visit** (post-visit remark, event photos upload, student CSV + girl/boy split bar, feedback+acknowledgement email, food/travel vendor-invoice stubs) · Timeline. Aside: Edit · Send email · Send feedback form · Download invoice · Mark completed.
- **Emails** (`spComposeEmail`/`spLogEmail`/`spBackfillEmails`, kinds `ack`/`confirm`/`fp`/`invoice`/`feedback`/`edit`/`status`): acknowledgement on enquiry, confirmation to the NGO + FP to staff on create, invoice to the **sponsor** on completion, feedback+acknowledgement form after visit — all viewable via the same Shopify-style **View email** modal. Like School, an **edit** save opens a "Save changes?" dialog (`spSaveConfirm`) with an "Email the POC" checkbox (`spCommit(notify)`) so any save can skip its email; the same dialog also gates the create confirmation email.
- **Not built (wireframe stubs)**: NGO/birthday-conflict calendar, reception educator-entry, eSignature, CSV import (upload derives a sample split), vendor-invoice files, blackout enforcement on the preferred date.

---

### 14. Merged `~/Downloads/muso-admin-portal.html` (Careers "Portfolio link")
3-way `git merge-file` (base `ab59f09`, ours HEAD `91794b6` = base + School/Sponsored, theirs = base + a Careers portfolio feature). One conflict — their reworded `career` schema sat where my new masters were inserted; resolved by taking **their** career schema (incl. the new `["Portfolio link|portfolio","bool",…]` field) then keeping **my** masters (fboptions…sponsored) after it. Their careers additions (portfolio bool on roles, portfolio URLs + `qual`/`exp` on applications, `CAREER_WATCH`/`CAREER_PH`/`tipRich` helpers, portfolio column + sort in the applications table) merged cleanly. Verified: `node --check` + sweep (0 failures across 28 masters) + live checks (careers portfolio row/field, School e-invoice, Sponsored invoice all render). Result is uncommitted in the working tree.

### 15. Masters/records UI pass — Edit buttons, column hygiene, Active toggles (latest session)
All verified with `node --check` + DOM-stub render sweep (0 failures) + targeted logic/render tests; browser attach was unavailable (a Chrome instance already held the chrome-devtools profile), so live checks were skipped this pass.

**Generic Foundations row Edit**
- Added a reusable master-row edit path: `let mEdit` (row index), `masterData(k)` (key → seed array: categories→CATS, tax→TAXRATES, hsn→HSNCODES, shopcats→SHOP_CATEGORIES, progcats→PROGCATS, fboptions/schoolslots/entryprices/sponsors/districts, pools→[POOLS.shared]), and `masterEditValues(k,i)` (maps a seed row → schema field names; special mappers for tax/hsn/pools/categories/progcats/shopcats, raw row otherwise, `is_active` defaults active).
- Render branch: `tab==="edit" && mEdit>=0 && masterData(current)` → `createForm(schema,label,"",masterEditValues(...),true)`. Handlers: `data-medit` (open), `data-medit-save` (return to list — display-only persistence, consistent with generic create). `mEdit` reset on nav / `data-goto` / New / any `data-tab`.
- Added a per-row **Edit** button to all 11 Foundations list views (categories, pools, tax, hsn, shopcats, progcats, fboptions, schoolslots, entryprices, sponsors, districts). Departments already had its own Edit/Delete.

**Tax view was dead code (fixed)**
- `M.tax.view` pointed at `"taxrates"` which has no `VIEWS` function → tax silently fell back to the generic `emptyList` (a 2-col Name/Status table). Re-pointed `M.tax.view = "tax"` so the real `tax()` view renders (now with the Edit button). Also fixed a latent bug there: it referenced a non-existent `r.when`; the "Charged when" text is now built from the seed's `field/op/val` (or "Always").

**Column hygiene (remove data with no matching form input; add Status everywhere it applies)**
- Removed columns that displayed values not captured by any form field: **Notes** (categories), **Draws on it** (pools — its `included_functions` field was removed from the form), **Component / Charged when / From / To** (tax), **Unit** (hsn).
- Added a **Status** column (reads `is_active`, defaults Active) to every Foundations list that has the field. **Departments** later got one too (see below). Kept legitimate derived count columns (Products, Roles on it, Bookings).
- **Inventory pools**: removed the **Excluded** column and the "concurrent heads" unit label; the pools **Edit** form now shows only **Basics** (code/name/capacity) + **Status** — the wrongly-labelled generic "Schedule & booking" section (capacity_basis / parent_pool / included_functions) was removed (new `current==="pools"` branch in `createForm`).

**Products & add-ons**
- Removed the **"Is inventory taken from the Building Pool?"** question and its now-empty **Inventory** section from the product form (kept on Programs — `inventoryFields()` is shared).
- Removed the in-list **Active toggle switch** from the Actions column (the only page that had one) + its dead `data-toggle-active` handler. Actions is now View / Edit; the Status pill still shows active/inactive.

**Blackout dates — Create working + seeds**
- `BLACKOUT_DATES` records seed (5: Independence Day, Gandhi Jayanti, a product-only Mini Golf closure, Christmas, Republic Day). `BLACKOUTS` (whole-museum dates) derives from it; `isBlackout` scans it live. New `VIEWS.blackout()` list (`M.blackout.view="blackout"`). The **Create** button is wired (`data-blackout-create` in the `createForm` footer + handler using `readForm()` → pushes a record). NB: the date field is a native picker, so a newly-created row shows an ISO date vs the seeds' `DD Mon YYYY`.

**Members**
- Renamed the list column **Status → Membership Status**; added a per-row **Edit** button. The edit form already existed (`memberForm(true)` pre-fills + "Save changes"); the bug was that the Edit (and **Renew**) buttons were swallowed by the row's `data-open` (View) handler. Fixed by intercepting `data-member-edit` and `data-open-renew` **before** the `data-open` handler (same pattern as the School/Sponsored edit-row buttons).

**Active switch replaces Status dropdown**
- **Membership tiers** form and **Shop coupons** form: the Status/Active-Draft dropdown is now an **Active** toggle switch. Coupon schema field `status` (enum) → `is_active` (bool); coupon edit-form values and seeds carry `is_active`. Generic `bool` field now defaults `is_active`/`active` **On** for new records (`val===undefined` → on) — applies to every create form.

**Shop coupons — Send for Review**
- Coupon detail: **Draft** coupons show a **Send for Review** button (`data-shop-coupon-review` → Pending approval); Pending shows **Approve coupon** (→ Approved); Approved shows neither. The approval **status** (Draft/Pending/Approved) and the new **Active** on/off toggle are now two separate concepts on a coupon.

**Departments — Active toggle**
- Added an **Active** field to the schema + seeds (all active), an **Active** toggle to both the create form and the rename/edit flyout (`deptEditLayer`/`deptSave` read it), and a **Status** column to the list. `deptCreate` reads the toggle.

### 16. Product Validation rules tab + Tax-rate cleanup (latest session)
All verified with `node --check` + DOM-stub render sweep (0 failures) + in-scope logic/render tests. Live browser checks skipped (chrome-devtools couldn't attach — a Chrome instance held its profile).

**New "Validation rules" tab on the product detail** (`SUBS` + `SUB.rules(a)`)
- Per-product entry rules on `P[i].rules = {product:[…], addon:[…]}` (display/config only — the POS/booking flow is **not** gated on them). A rule is `{subject, requires:<category|category[]>, mode:"any"|"all", min, max}` meaning *"for each `subject` ticket, require min–max of the required category(ies)."* `min 1` = mandatory; `min 0` = may enter alone; blank `max` = no limit.
- **OR/AND support:** `requires` can be **multiple categories** with an **Any of (OR) / All of (AND)** match mode → e.g. "Every Adult needs 1–5 Child 3–11 **or** 1–5 Child 12–17."
- Tab UI columns: **For each** (select) · **Match** (always a `needs any of / needs all of` dropdown) · **Requires** (category chips, toggle to include) · **Min** · **Max** · **In plain words** (live `ruleSentence`) · **Remove**. `+ Add rule` + empty state. Helpers `ruleSentence`/`ruleProblem`/`ruleReqs` (reuse `ratioInt`); category options from `a.cats`.
- **Remove → confirm dialog** (`ruleDel` state + `.confirm-layer`): "Remove this rule?" quoting the rule; `data-rule-del` opens it, `data-rule-del-confirm`/`-cancel` resolve; `ruleDel` cleared on nav + `data-sub` switch.
- Handlers: `data-rule-add` (seeds `{subject, requires:[firstOther], mode:"any", min:1, max:""}`), `data-rule-req` (chip toggle — placed **before** the generic `.chip-opt` handler so it intercepts), `data-rule-del`/`-del-confirm`/`-del-cancel`; `data-rule-field` (change) writes subject/requires/min/max/mode.
- **Contexts:** originally rendered "As a product" + "As an add-on" cards; the **add-on section was removed** — now a single **Entry rules** card (product context) per product.
- Seeds on the four products (MuSo Tour, Mini Golf, Lost Code of Play, Vroom): (1) every Child 3–11 needs 1–5 Adult; (2) every Adult needs **any of** 1–5 Child 3–11 or 1–5 Child 12–17 (Child 12–17 comes alone).
- **Removed the old form ratio widget** (`ratioField` in `productForm`'s booking array — it was form-local and never saved); its helpers stay for reuse. `rebuildRatio`/`syncRatio` are now no-ops (guarded on a missing field).

**Tax rates removed from Products & add-ons** (they belong to the HSN/SAC code, which already carries them)
- Removed the **"Tax rules" tab** from the product detail (`SUBS`), the **Tax-rates multi-select** from the product form (`finance` now = HSN/SAC only; section renamed **"Tax / invoice code"**), and the **"Tax rates" row** from the Details tab's Tax section — which now shows just the **HSN / SAC code** plus the rate it inherits (e.g. `SAC 999692 · 18% (via the code)` via `hsnTotalPct`). The `P[i].tax` seed array is **left intact** so downstream invoice/GST math is unaffected; only the product-facing UI stopped duplicating it.

### 17. Capacity blocks per-slot quantity, upstream merge, Status→Active swaps (latest session)
All verified with `node --check` + DOM-stub render sweep (0 failures) + in-scope logic/render tests. Live browser attach still unavailable.

**Capacity blocks — per-time-slot blocked quantity** (modeled on the Discounts "Quantity per item" flyout)
- A block now carries `qtys:{ [slotName]:number }` instead of a scalar `qty`; `slot` stays as the joined blocked-slot names. Back-compat helper `blockQtys(bk)` reads either shape.
- The block flyout (`blockFormCard`) gained a **searchable time-slot multi-select** (`blockSlotSelect`, msel — mirrors `itemSelect`); **only selected slots** appear below as rows (`slot · time` + its own `blocked` input). A `blockDraft` working copy + `captureBlockFlyout()` preserve dates/reason/typed amounts across the toggle re-renders (mirrors `condDraft`/`captureCondFlyout`). State: `blockDraft`, `blockMselOpen`; handlers `data-blockslot` (toggle, before the generic `.chip-opt` handler) and `data-blockmsel-toggle`; draft seeded in the `data-edit-block`/`data-addopen="block"` handlers and cleared on cancel/save.
- `SUB.availability` Slot + Blocked columns render per-slot (aligned multi-line). `addSubmit` block branch reads the draft. Seeds migrated to `qtys`, plus one multi-slot demo block on MuSo Tour (Three-hour morning −10, Five-hour morning −15).

**Upstream merge** — pulled `origin/main` (fast-forward `a1f861b..18da552`, RajShah2022 "refactor: drop tax conditions and category requires rule") over stashed local work; **stash popped with no conflicts**. Their refactor is **complementary** to ours: they removed the "requires another category" rule from the **visitor-category master** (dropped `req`/`reqMin`, `needsOf`/`needsText`/`UNACCOMPANIED`, the "Must be booked with" column + its schema fields `requires_category_id`/`requires_min`, and simplified the `"Tax rates"` OPTIONS + categories create-form/`masterEditValues`), pointing that rule "to the product/listing" — which is exactly our **Validation rules tab**. Post-merge verified: 0 conflict markers, 0 stale refs to the deleted helpers/fields, all our features present, sweep clean.

**Status dropdown → Active toggle (create forms)**
- **Product form** (`productForm`): removed the Status enum dropdown (`STATUS_FIELD`/`PROD_REST[4]`) from Basic information and put the **Active** switch in its place (`activeField`, seeded from `values`); removed the standalone bottom **Status** section.
- **Program form** (`createForm` `label==="program"`): removed `"status"` from the Basic-information field list, leaving the existing `is_active` **Active** switch in its place. (No separate Status section for programs.)

### 18. Venues master + venue-availability + booking-form parity (latest session)
All verified with `node --check` + DOM-stub render sweep (0 failures) + live chrome-devtools checks (the browser was attachable this session).

**Venues master (Foundations, step 13)** — sourced from `~/Downloads/Rentals.docx`
- `VENUES` seed (10 rentable spaces: Field Area, Commons, Art Gallery, Connect Lab, Small MPA, Library, MPA, Classroom, Green Room, Grow Lab) with name/floor/size/seating/floating/is_active. Registered via `ORDER`/`PHASE`/`PREREQ`/`M`/`SING`/`masterData`; `VIEWS.venues()` list + per-row Edit (generic `data-medit` path).
- The School/Sponsored visit **Optionals (venue blocks)** "Venue" free-text became a **dropdown** sourced from active venues (keeps a legacy value not in the master). The single **Time** field split into **Start Time / End Time** (parses/rebuilds `o.time` as `start–end`).

**Venue availability** (a venue is shared by **Events and Experiences**, School, Birthdays — the module formerly called "Rentals" was renamed to "Events and Experiences", incl. `VENUE_BOOKINGS.module` labels + the Venues master owner/src; the §4 inventory-pool exclusion list keeps its own "Rentals"/"Events"/"Workshops" taxonomy)
- `VENUE_BOOKINGS` seed = sample cross-module occupancy; `venueDayBookings(venue,date,src,skip)` = seed rows + the draft's own other same-venue/date optionals; `timeToMin`, `venueClashCheck`.
- **"View availability"** button (per optional, before Start Time) opens `venueAvailCard()` — a drawer flyout with **Venue + Date filters** (venue dropdown auto-applies + carries the typed date; **Apply** button commits a typed date) and a **30-min time-table 09:00–19:00** showing booked slots greyed with module · who · purpose (no "…continued"). Header is just "Availability". Filters prefill from the block. State `venueAvail`; handlers `data-school-viewavail`/`data-sp-viewavail`, `data-venueavail-venue`/`-date`/`-apply`/`-close`. **NB:** `scrub()` (line ~5876) strips whole `.note note-scope`/`note-open` blocks — the flyout's banners use plain inline-styled divs, not `.note`.
- **"Check availability"** button (per optional, in the Start/End row) — inline verdict (`runVenueCheck` → `optCheckBanner`): red "Clashes with …" / amber "Fill … first"; disabled until Venue+Date+Start+End filled (live-toggled by `syncOptCheckBtn` on input/change), and any edit clears a stale verdict. State `venueCheck`.
- **Create/Save is blocked on a clash**: the "Create booking?/Save changes?" confirm dialog runs `bookingClashes(src)` across all venue blocks — a red clash list disables the confirm button (guarded again in the commit handler); no green "all-clear" box; the create copy was lengthened. Optional-box layout is now two 3-col rows: **Venue · Date · View availability** and **Start · End · Check availability**.

**Careers — Portfolio (Applications)**
- `portfolio` on an application is now **free text from a frontend textarea**, shown **plainly like the Notes column** (no link parsing, no click-to-open). `appPortText(a)` normalizes string-or-array; the role toggle copy → "Portfolio links … a textarea". The empty-but-asked state shows a muted **—** (was "Left blank").

**Sponsored School — parity with School** (all forms/handlers mirror the school ones)
- Added: **Other** chip in Education Board (`SP_BOARDS` gained "Other" → the conditional "Other Board" field, already wired), an **Additional Information** section (Message/Note, captured via `note`), a **Billing (for invoicing)** section after Visits (billName/billAddress/gstin/pan, captured; GST hint notes the sponsor invoice is against the sponsor).
- **F&B options** (per-visit, both School & Sponsored) changed from multi-select chips to a **single `<select>`** (`data-vfsel`/`data-spvfsel`; `optionIds` holds 0/1).
- **Underprivileged children remarks** (Impact & remarks) changed from a textarea to **repeatable Title+count rows** (`underprivileged` now `[{title,count}]`; `data-sp-uprow`/`data-spf-up`, add/remove handlers, seeds migrated, detail renders "Title: count · …").
- Confirmed **post-visit event-photo upload** already exists on the Sponsored detail's **After visit** tab (`postVisit.images`, `data-sp-upload="images"`) — Sponsored-only.

**Products — Tax** (from the merged upstream direction): removed the product's own tax-rates UI — the **Tax rules** tab, the form's tax-rate multi-select (section now "Tax / invoice code", HSN/SAC only), and the Details tax-rates row (shows HSN/SAC + the inherited rate). `P[i].tax` seed kept for downstream math.

---

### 19. DB-decision follow-ups (2026-09-24)
Owner answers from the fastapi-boilerplate schema Q&A, applied to the mockups:
- ~~**One discount per order (Q14=B).** POS summary picks the single biggest saving among the typed code, every automatic discount and the member add-on perk; the rest are listed struck through as "not applied". Replaces the old "stack all".~~ — reversed same day, see stacking below.
- **Discount stacking (owner, 2026-09-24).** Every discount has "Can be combined with other discounts" (`d.combinable`, form checkbox `data-combinable`). POS takes several codes (`bookingDiscountCodes`, chips with ×). Summary applies the bigger of: all combinable discounts stacked (% first, then flat, each on what is left) vs the best non-combinable one alone; the rest show struck through with the reason. Seeds: bundling, Membership, Members-programs, JSW series combinable.
- **"Underprivileged" → "Sponsored" (2026-09-25).** Sponsored booking field `underprivileged` renamed `sponsoredGroups`; labels "Sponsored children remarks" / "Sponsored children".
- **State dropdown (Q15=B).** `IN_STATES` / `STATES`; POS customer step (`bookCustState`, default Maharashtra) and website "Your details". Place of supply — Maharashtra = CGST+SGST, else IGST; GSTIN prefix overrides.
- **Names only for programs (Q18=A).** POS step 3 and website details page ask names only for events/camps/workshops, plus the parent's WhatsApp number. Tours, add-ons and membership visits are counts only.
- **Member perks live in Discounts (Q4=A).** Plan seeds lost `discounts:{birthday,events,workshops}`; plan list/detail/form and member detail show `tierPerks(t)` — discounts with the membership condition or members-only flag. Members-only discounts get a "Which plans" chip picker (`d.memberTiers`, none = every plan).
- **Plan HSN (Q5=A).** Plan form + detail carry an HSN / SAC code (`t.hsn`, default 999692).
- **Careers (Q20=A).** Application `note` split into `msg` (candidate's message) and `hr` (HR notes, HR-only).

## Known wireframe simplifications / open notes
- Auto discounts apply on **simplified bases** (whole-cart or the obvious line group), not a full `target/prods/cats` engine.
- ~~Per "stack all", there is **no** mutual-exclusion (`no[]`) or biggest-saving enforcement in the POS — a member can get both the member add-on line and the Membership auto row.~~ — superseded by §19: one discount per order
- `recalcBookingSummary()` / `renderBookingAttendees()` remain **dead code** (key off non-existent `[data-bookqty]`) — left untouched.
- Seeds still carry some inert legacy fields (`prec`, `no`, old `apply` strings) — harmless.
- Git HEAD is `042a3c2` (branch `main`, "Remove the Status dropdown from Products & add-ons & Programs…") — sections 12–17 committed; up to date with `origin/main`. **All of section 18 (Venues master, venue-availability, Careers portfolio, Sponsored parity, product-tax cleanup) is uncommitted** in the working tree (`muso-admin-portal.html`), plus this `CONTEXT.md` update — commit when ready. Untracked docs (`VENUE-MODULE-PLAN.md`, scope docs, `.xlsx`, etc.) remain untracked.
- **Remote is now HTTPS** (`https://github.com/aayushah711/muso-wireframes.git`, switched from SSH): on this network port 22 is blocked and SSH-over-443 is dropped by a middlebox (`20.207.73.82`), so fetch/pull/push must go over HTTPS. An unused `~/.ssh/config` 443 route + GitHub host keys were added (backed up as `~/.ssh/*.bak.*`).

## Quick reference — key seeds & helpers
- Seeds: `P` (products/add-ons — each with `rules:{product,addon}` for the Validation-rules tab and `blocks[]` with per-slot `qtys`), `BOOKINGS`, `CUSTOMERS`, `DISCOUNTS`, `MEMBERSHIP_TIERS`, `MEMBERS`, `CATS` (visitor categories — **no longer** carry `req`/`reqMin`; pairing rules moved to the product), `WORKSHOPS`, `EVENTS`, `PROGCATS`, `BLACKOUT_DATES` (records; `BLACKOUTS` = whole-museum dates derived from it), `SHOP_COUPONS` (carry `status` + `is_active`), `DEPARTMENTS` (carry `is_active`), `VENUES` (rentable spaces master), `VENUE_BOOKINGS` (sample cross-module occupancy for availability), `DASH`, `LOCATIONS` (none — Location is free-text). Sponsored records: `underprivileged` is now `[{title,count}]`; School/Sponsored visit `food.optionIds` holds 0/1 (single-select).
- Availability helpers: `venueDayBookings`, `venueClashCheck`, `bookingClashes`, `runVenueCheck`, `timeToMin`; flyout `venueAvailCard` + state `venueAvail`/`venueCheck`.
- Discount code that works in POS: **IND50, MUMKIDS150, JSW001–JSW100** (all now), plus auto discounts by cart condition.
- Backups this session: `/tmp/pre-multislot-backup.html`, `/tmp/pre-merge-backup.html`.
