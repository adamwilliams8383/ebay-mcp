# Vintage batch → Amanda

The own-stock branch of `route-orders`. Policy is `ROUTING_RULES.md` §7; UI mechanics are in
`ROUTING_TECHNIQUES.md`. This file is the working order of operations.

## When this fires

From Step 3 classification, either:

- green `N In stock` > 0 and the "Your warehouses" modal reads **Vintage Direct Upload,
  Beaver Dam WI**, **and** the SKU is in today's inventory file; or
- an **always-Vintage brand** (Daimler, Paccar, Harley-Davidson, Royal Enfield, John Deere,
  AGCO, Kubota, CNH, Spartan/Shyft, UD Trucks, Oshkosh — *not* Continental) showing 0/0, where
  the SKU **is** in today's file.

If the SKU is **not** in the file, stop: note `OUT OF STOCK AT VINTAGE - ADAM`, report the
order ID, and let Adam take it. Do not build a VP row, do not assign in ShipStation, do not
run rule A.

> A large negative RP "Net Profit" on a Vintage order is **normal** and is not a rule C
> trigger — it is RP's dropship math on an order it has no supplier cost for. Real margin
> comes from the inventory file. Rule C applies only where a supplier cost exists.

## 1. Cost — the inventory file is the only source

```bash
python S:/CLAUDE/automation/inv.py "<FULL eBay Custom Label>"
```

Today's file is `S:\CLAUDE\Inventory File YYYY-MM-DD.xlsx`, dropped ~08:00. **Verify the cache
came from today's file before trusting a cost** — on 09-01 a stale prior-day pickle was nearly
used on a live send. Columns: `OEM_NUMBER, OEM_NAME, PART_NUMBER, PART_DESCRIPTION, QTY,
PRICE, WEIGHT`; ~690,000 keys.

Look up the **full Custom Label and the stripped part number — either can be the hit**:

```
base   = SKU minus a trailing brand word
nodash = base with dashes removed
EX[sku] -> NM[nk(sku)] -> EX[base] -> NM[nk(base)] -> EX[nodash] -> NM[nk(nodash)]
```

Brand words: gm, mopar, ford, toyota, honda, kia, hyundai, subaru, nissan, acdelco, mazda,
dorman, delphi, mercedes-benz.

The file `PRICE` is the **COST**, per unit.

## 2. Custom Label for the file

Read the Custom Label off the **eBay order detail page**, not RP's SKU field:

```js
document.body.innerText.match(/Custom label[^\n]*\n[^\n]*/gi)
```

- Strip the trailing brand word.
- **FORD and MAZDA also strip dashes**, and both transforms apply together. Live case:
  eBay `N0Y7-26-46Z mazda` → RP shows `N0Y7-26-46Z` → the VP file needs `N0Y72646Z`, which is
  what the inventory file's own `PART_NUMBER` carries.
- Non-automotive Vintage SKUs stay exactly as-is, dashes included.

## 3. File name — check Proton Sent first

`ORDERS MM-DD-YYYY VP<n>.xlsx`, where `<n>` is the next free number for **today**.

**Always check Proton Sent for today's already-sent VP numbers before naming the file** —
other runs send files too, and numbers collide silently.

## 4. Build it

`build_vp.py` owns the formatting (Aptos Narrow 11, bold header, C–G centered, D–G
`$#,##0.00`, sheet `Sheet1`, fixed column widths):

```bash
python S:/CLAUDE/automation/build_vp.py spec.json
```

```json
{"date":"09-08-2026","vp":3,"rows":[
  {"order":"12-34567-89012","label":"N0Y72646Z","qty":1,"sold":69.92,"cost":47.80,
   "promoted":false,"shipping":0.0,"name":"...","phone":"...","addr1":"...","addr2":null,
   "city":"...","state":"..","zip":"...","title":"..."}
]}
```

Columns: Order Number, Custom Label, Quantity, Sold For, COST, -35%, Shipping And Handling,
Ship To Name, Ship To Phone, Ship To Address 1, Ship To Address 2, Ship To City,
Ship To State, Ship To Zip, Item Title.

- **`sold` and `cost` are BOTH PER-UNIT.** `sold` = the eBay **subtotal**, not the order total.
- **`promoted`** drives the `-35%` column: `cost * 0.55` when the eBay order detail "Selling
  costs" shows an **AD FEE** line (promoted listing); `cost * 0.65` when it shows transaction
  fees only. The script computes it — set the flag, don't precompute the value.
- **Shipping And Handling:**
  - free-shipping automotive → RP's `Shipping: $X USD (included in price)`
  - buyer-paid → the eBay amount
  - heavy-duty truck (Daimler/Paccar) `$0.00` stays `$0.00`
  - multi-item → sum the per-item shipping onto the parent row

**Multi-item layout:** a parent row (order number, blank Custom Label, total qty, total Sold
For, blank COST and -35%, summed shipping, ship-to fields, blank Item Title), then one child
row per item (blank order number, Custom Label, per-item qty, per-unit Sold For, COST, -35%,
blank shipping and ship-to, Item Title).

## 5. Then, in this order

### a. eBay note on EVERY line item

```bash
node S:/CLAUDE/automation/note.js <ORDER#> "SENT TO AMANDA" --all-items
```

`--all-items` is required on a multi-item order (`ROUTING_RULES.md` §6: "Multi-item orders
get the note on EVERY line item"). Such orders have **no order-level note entry at all** — the
note lives on the per-item kebabs — so a plain `note.js` call covers only one item.

The flag asks eBay how many line items the order really has, writes, then re-counts the
matching notes on the page and advances until the counts agree. It does not assume an
index-to-item mapping, because the kebab list can include an order-level kebab that carries no
note entry. It exits non-zero if it cannot reach every item, so a partial run is visible
rather than silent.

### b. ShipStation — assign to Amanda Janz

```json
shipstation_assign_orders {"params":{"order_numbers":["..."],
  "user_id":"8f0dad38-d327-4b73-b29a-29c8e6ca0b43"}}
```

`not_found` means the store has not synced. Fix it with:

```bash
node S:/CLAUDE/automation/ssrefresh2.js --update   # clicks refresh -> Update All, waits 25s
```

Then retry the lookup. Verified 2026-09-08: every recent order was missing, including one
routed hours earlier; after `--update` they all resolved.

**Do not hard-code coordinates** — the refresh control is a bare `<svg>` with no aria-label,
title, or `<button>` ancestor, found by its circular-arrow path data (`M512 192l0-200`) in the
top nav. Positions move with window width (1414,19 in an older note vs 1387,25 today); the
script re-locates it every run.

One order number can return **two** ShipStation records — an `awaiting_payment` one and an
`awaiting_shipment` one, sometimes with different SKU spellings. Assign the
awaiting_shipment record; `shipstation_assign_orders` already picks it.

If the sync still does not surface the order, **send the email regardless** — a sync lag must
not hold up Amanda's file — and report the assignment as outstanding.

### c. Proton email

- To `ajanz@vpartsinc.com`, subject = **the exact file name**, body `Regards, Adam Williams`
  (Proton's signature already supplies this), attach the file.
- Before opening "New message", check whether a composer is **already open** with that
  subject and reuse it — reloading the mailbox auto-saves composers into Drafts and you will
  stack duplicates.
- Compose: real-click "New message" (may need two). Real-click the To field and type the
  address; then `.focus()` the subject field via JS and type. **Do not click blindly at the
  subject's y-coordinate** — it lands on the To field's second row and files the name as a
  second recipient.
- Attach with `setInputFiles` on `input[type=file]`; no coordinate clicking.
- Proton **replaces the typed address with the contact display name**, so the composer reads
  "Amanda Janz", not the raw address. Accept either form in a pre-send check.
- Screenshot and confirm To / subject / attachment before Send.

### d. Verify it actually sent

`https://mail.proton.me/u/1/all-sent`. **"Servers are unreachable" means the send failed
silently into Drafts** — reopen the draft and send again. The Drafts list re-sorts, so confirm
the draft's date before opening it. Never report a Vintage batch as sent without seeing it in
Sent.

### e. Hand off

Deliver the file to Adam in chat and save a copy to `S:\CLAUDE`.

## Recording

Note text is `SENT TO AMANDA` (`ROUTING_RULES.md` §6). Record each order in
`.claude/state/routed-orders.json` with `status: "vintage"`, the VP file name, and the
per-unit cost, then update `S:\CLAUDE\ROUTING_STATE.md` at end of run.
