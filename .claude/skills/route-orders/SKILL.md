---
name: route-orders
description: Route new eBay orders to a Revolution Parts warehouse. Pulls unfulfilled eBay orders, looks each part up in Revolution Parts, picks the warehouse using routing-rules.md, places the order, and hands out-of-stock lines to stock-sync-guard. Use when asked to route orders, check for new orders, process the order queue, or run the order sweep.
---

# Route eBay orders to Revolution Parts

This is the Claude Code replacement for the Cowork auto-routing browser flow. One order
at a time, end to end, with the money-spending step gated behind confirmation.

## Before you start

Read these first. Do not improvise around them.

- **`S:\CLAUDE\ROUTING_RULES.md`** — the business policy, and the final word. Adam owns it and
  edits it freely; read it fresh every run. `.claude/skills/route-orders/routing-rules.md` is
  only a pointer and a summary.
- **`S:\CLAUDE\ROUTING_TECHNIQUES.md`** — accumulated UI mechanics for the same screens.
  `revolution-parts.md` in this folder is a distillation; that file is the living source.
- `.claude/skills/route-orders/revolution-parts.md` — the short form of the above.
- `.claude/skills/route-orders/vintage-batch.md` — the own-stock branch: inventory lookup,
  the VP xlsx, ShipStation and the Proton send to Amanda.
- `.claude/state/routed-orders.json` — local routed cache. Read with
  `cat .claude/state/routed-orders.json`; if it does not exist, treat it as `[]`. It is a
  cache, never the source of truth.
- `S:\CLAUDE\ROUTING_STATE.md` — live per-order state from previous runs.

Most of this flow is already implemented in `S:\CLAUDE\automation`. Prefer those scripts over
driving the browser by hand — they carry guards earned the hard way.

| Script | Use |
| ------ | --- |
| `scan.js` | list eBay awaiting-shipment orders, split noted vs un-noted |
| `rp.js <ORDER#>` | read-only RP order dump |
| `carvcheck.js <RECORD#>` | Carvana screening evidence |
| `route.js <RECORD#> [--commit [supplier]]` | dry-run / commit the dropship route |
| `oos.js <ORDER#> [--commit]` | rule §4A out-of-stock workflow |
| `gideon.js --add <ORDER#> --send-if-new` | rule §4B batch to Gideon at RP |
| `note.js <ORDER#> "TEXT" [--all-items]` | write the eBay seller note; `--all-items` for multi-item orders (§6) |
| `inv.py "<SKU>"` | Vintage inventory-file lookup |
| `alert.js "<subject>" "<body>"` | **reach Adam from an unattended run** — SMTP, not push (push silently no-ops) |
| `build_vp.py <spec.json>` | build the `ORDERS MM-DD-YYYY VP<n>.xlsx` for Amanda |

## Step 0 — Reconcile rule §4B orders first

Gideon fixes no-route orders on the RP side, sometimes within the hour, and nothing writes the
result back to eBay. A stale `CANT ROUTE ORDER CONTACT GIDEON` note hides an order that is
actually routed, and makes the next run treat a filled order as a problem. Close that loop
**before** scanning for new work.

**Timing.** Gideon normally comes back within about an hour of the first batch email
(Adam, 2026-09-08), so the hourly sweep is the right cadence and the run an hour after a batch
is the one most likely to find fixes.

Do not wait for his reply, and do not gate the check on it. He does not always reply, and the
PO appearing in RP is the real signal. Reading the reply changes nothing — the active PO still
has to be confirmed in RP before any note is rewritten — so the mailbox is at best a hint that
it is worth looking sooner, and is not worth a Gmail dependency.

1. List every awaiting-shipment order whose note is `CANT ROUTE ORDER CONTACT GIDEON`
   (`lib.scanUnnoted` returns `.note` on every row).
2. Re-read each in RP and count **non-cancelled** supplier orders.
3. Then:

| Active POs | Meaning | Action |
| ---------- | ------- | ------ |
| 0 | Still stuck | Leave the note. **Do not re-queue to Gideon** — he already has it and the queue clears on send. Zero POs is **not** rule §4A either; the part is still in stock. |
| 1 | Fixed | Replace the note with `ROUTED TO <SUPPLIER> PO <n>`. Take the supplier from the Supplier Orders panel's **Dealership** column, never the Warehouse field. Report it. |
| 2+ | Duplicate | Fix per `ROUTING_RULES.md` §1 first, then write the note for the surviving PO. |

`rp.js` prints only the PO number; the dealership is on `lib.getRpOrder(...).supplierOrders[].dealership`.

First pass on 2026-09-08 found 3 stale notes out of 8 — two fixed within the hour of the batch
email, one (19-15116-50053 → PO 1881 Sophio) stale since 09-06.

## Step 0b — Warehouse-cancelled orders (rule §4E)

A warehouse can back out after accepting. Those orders sit in RP as
**In Progress / Cancellation Requested** and never appear in the un-noted scan, because they
already carry a routed note. Sweep them each run:

```
/customers/22764/orders/open?filter=order_status:In%20Progress;order_substatus:Cancellation%20Requested
```

**A cancellation is not automatically a dead order — try to re-route first.**

1. **Approve it.** On the order detail page set **Order Status → "Cancellation Accepted"**.
   That cancels the PO. Re-read and confirm the Supplier Orders row says Cancelled and
   ACTIVE=0 — until then both `route.js` and `oos.js` refuse the order as `ALREADY ROUTED`.
2. **Re-read stock.** The cancellation usually means that warehouse ran out. `0 In stock` and
   `0 In shared` → straight to step 4.
3. **Try to re-route** — `node route.js <RECORD#>` dry run, then commit under the normal §5
   rules. **Never re-route to the warehouse that just cancelled**; if it is the computed pick,
   name the next best instead: `node route.js <RECORD#> --commit "<other supplier>"`. On
   success, verify one active PO and note `ROUTED TO <SUPPLIER> PO <n>`. Done.
4. **Nothing left to route** → hand to `stock-sync-guard` for the rule §4A actions (quantity
   to 0, buyer message, note `MESSAGED TO CANCEL MM-DD`).

Never run the rule §4A actions first and re-route afterwards — messaging a buyer to cancel an
order you could still fill throws the sale away.

## Step 0c — Buyer-requested cancellations (rule §4F)

RP files these in the **same** In Progress / Cancellation Requested bucket as rule §4E, so the
Step 0b sweep surfaces both. They are different rules with different actions.

**Tell them apart by who asked.** The order detail says plainly
*"The buyer has requested a cancellation: Approve | Reject"* — that is §4F. Rule §4E is the
**warehouse** backing out and shows a **cancelled supplier PO**; a §4F order usually has no
supplier order at all.

**This is not rule §4A.** Do **not** zero the listing and do **not** message the buyer. The
part is not out of stock — the listing should keep selling — and the buyer has already asked,
so asking them to ask is nonsense.

**Never approve or reject it yourself** (§0: Adam cancels on his side).

| State | Action |
| ----- | ------ |
| **Not routed** (no active PO) — the normal case | Do not route. Note `BUYER REQUESTED CANCEL - ADAM TO APPROVE`. Report the order ID. Nothing else. |
| **Already routed** (active PO) — the expensive case | Do **not** cancel the PO (§0 does not permit it for this). Note `BUYER REQUESTED CANCEL - PO <n> ALREADY PLACED - NEEDS ADAM` and alert Adam with `alert.js` (**not** push) giving supplier, PO and what we paid. |

Once noted, the order is done from our side. It keeps appearing in the Cancellation Requested
filter until Adam approves it — that is expected, not a new problem.

## Step 1 — Pull the unrouted queue

Call `ebay_get_orders` with:

```
filter: "orderfulfillmentstatus:{NOT_STARTED|IN_PROGRESS}"
limit: 50
```

Drop any order with a non-empty `cancelStatus.cancelState` other than
`NONE_REQUESTED` — a cancel is already in flight; report it, don't route it.

Then apply **all three** already-routed checks below, per `ROUTING_RULES.md` §8. Local state
alone is not sufficient: it is empty on a first run, it knows nothing about orders handled by
any other operator or tool, and it goes stale the moment a run is interrupted.

### 1a — The eBay seller note (primary marker)

An order carrying a seller note has already been handled by some run — routed, batched to
Vintage, or parked on an exception rule. Skip it. The whole cycle is defined as *"scan eBay
awaiting-shipment for orders with NO seller note"* (`ROUTING_RULES.md` §2). Note text and
meanings are listed in §6 of that file.

### 1b — Local state (fast, advisory)

Drop any order whose `orderId` appears in `routed-orders.json` with `status: "routed"`.

### 1c — RevolutionParts is the source of truth (authoritative)

For every order that survives 1a, confirm in RevolutionParts that no supplier order exists
before routing it. Open the RP order (Selling → Orders → Open → **New**; the eBay order
number is the `Order #` column, and `Record #` is the RP id used in
`/customers/<customerId>/orders/<recordId>`) and treat the order as **already routed** if
either of these is true:

- The **Supplier Orders** table on the order has any row. A row means a supplier order was
  placed — capture its `Order #` and dealership for the summary.
- The activity log contains a `Routed supplier order #… to supplier order #…` entry.

A line item reading `Dropshipped to order <n>` is the same signal at line level.

**Do not use the Warehouse column or the order-level Warehouse dropdown as the routed
signal.** A routed order can still show a blank Warehouse / "Not Assigned" — verified on
order `74870144` (eBay `24-15129-48762`), which had been routed to supplier order `1169`
while its Warehouse column was empty. Routing on that signal re-orders parts that are
already on their way.

Parse it the way `ROUTING_TECHNIQUES.md` does: slice `innerText` from `Supplier Orders` to
`Actions`, rows match `/^d{3,5}	/`, and a row is **active** unless it matches
`/Cancel(l)?ed/`. A Cancelled row does not count as routed.

When the checks disagree, RevolutionParts wins: record the order in `routed-orders.json` with
the supplier order number so later runs skip it cheaply, write the missing eBay note, and move
on without routing.

An order that is not in RevolutionParts at all is sync lag, not a problem — skip it, leave it
un-noted, and let the next run catch it (often ~10 minutes later).

Report the remaining queue as a short table (order ID, buyer, SKU, qty, ship-to state,
order date) before doing anything else. Say how many orders each check dropped. If the queue
is empty, say so and stop.

## Step 2 — Screen the buyer before anything else

`ROUTING_RULES.md` §4D. Carvana orders cost Adam money nearly every time, and the shipping
block never gives them away.

```bash
node S:/CLAUDE/automation/carvcheck.js <RECORD#>
```

Match everything it prints — the RP page, ship-to, buyer user id and email — against
`S:\CLAUDE\CARVANA WATCHLIST.txt`, read fresh per order. Then:

- **CONFIRMED** (watchlist hit anywhere) → do not route. Note `CARVANA ORDER PLEASE CANCEL`, report the order ID.
- **CLEARED** (a buyer Adam approved) → overrides everything, even a literal Carvana address. Route normally.
- **SUSPECT** (shape only — ship-to name ending in a long digit string, street line ending in a reference number, "inspection center") → HOLD. Note `POSSIBLE CARVANA - ADAM CHECK BUYER`. Never the CONFIRMED wording; this asks for a human look, not a cancel.
- **ERROR** (watchlist missing or has no CONFIRMED entries) → HOLD and tell Adam. An unreadable watchlist must never read as "no match".

Dealer ship-tos and freight forwarders (Aeropost, eIS, "Service Dept PO-xxxxxx",
"Pro Motion Racing") are **not** Carvana — route normally.

**Do not raise these — Adam ruled on them 2026-09-08.** They look like the shape §4D describes
and are not:

- **eBay international / Global Shipping (GSP).** A `Customer Shipping Method` of
  "eBay International Priority Shipping DDP" and **no shipping address block on the RP page**
  is a normal international order — the parcel goes to eBay's domestic hub, so there is no
  end-buyer address to screen. Carvana is a US inspection-center operation. Route normally and
  do not flag the missing block. (Seen on 26-15111-34700, a Polish buyer.)
- **Auto-glass and other trade buyers.** A buyer whose email is at an auto-glass or other
  trade business domain is a trade customer, covered by the dealer rule above. Route normally.
  (Seen on 01-15158-58929 and 06-15147-92523.)
- **Ship-to name ≠ buyer name.** An ordinary household or gift pattern — the ship-to is a
  different family member from the account holder. This is an eBay-side artefact, not our
  problem, and it is not the SUSPECT shape. SUSPECT means a trailing long digit string, a
  street line ending in a reference number, or "inspection center" wording — not a plain name
  mismatch. Route normally. (Seen on 17-15127-53485.)

## Step 3 — Classify from the RP order detail

```bash
node S:/CLAUDE/automation/rp.js <ORDER#>        # read-only dump
```

Read the **line-item inventory panel** (`ROUTING_RULES.md` §3). Never the Warehouse column.

| Signal | Meaning | Action |
| ------ | ------- | ------ |
| green `N In stock` > 0 | we own it — only Vintage and Pinnacle hold own stock | open the "Your warehouses" modal and branch below |
| green `0` + blue `N In shared warehouses` > 0 | dropship candidate | → Step 4 |
| `0 In stock` and `0 In shared warehouses` | nothing to route | → Step 5, rule A — **but check the trap first** |

**Own-stock branch:**

- *Vintage Direct Upload, Beaver Dam WI* → Vintage order. Look the FULL eBay Custom Label up in today's file: `python S:/CLAUDE/automation/inv.py "<SKU>"`. **In the file** → follow **`vintage-batch.md`** end to end (cost, Custom Label transforms, VP file, note, ShipStation, Proton). **Not in the file** → note `OUT OF STOCK AT VINTAGE - ADAM`, report the order ID, stop. Do not run rule A on it.
- *PINNACLE PARTS, Coral Springs FL* → on hold entirely. Note `IN STOCK AT PINNACLE - ON HOLD - NEEDS ADAM REVIEW` and tell Adam.

**The always-Vintage trap.** For Daimler, Paccar, Harley-Davidson, Royal Enfield, John Deere,
AGCO, Kubota, CNH, Spartan/Shyft, UD Trucks, Oshkosh — 0/0 with no suppliers is the NORMAL
presentation, not an out-of-stock signal. (Continental is **not** on this list.) Look the SKU
up with `inv.py` before treating it as rule A; on 2026-09-02 this distinction was worth
$759.56 on order 07-15119-33809. If it is in the file, it is a Vintage order — go to
**`vintage-batch.md`**, not rule A.

## Step 4 — Dry-run the route

**Always dry-run first.** This is where supplier cost and distance come from — there is no
cost-comparison screen in the RP UI.

```bash
node S:/CLAUDE/automation/route.js <RECORD#>          # DRY RUN, changes nothing
```

It prints every supplier offered (`supplier  $cost  miles  stockLevel`) and its own
`WOULD PICK`. The selection rule is already implemented in `lib.js` and matches
`ROUTING_RULES.md` §5 — cheapest first, closer wins when within ~$5, a $10+ gap always takes
the cheaper one.

Report the offered suppliers and the pick before acting. Handle a refusal rather than working
around it:

| Refusal | Meaning | Action |
| ------- | ------- | ------ |
| `ALREADY ROUTED` | an active PO exists | stop — write any missing eBay note, record it, move on |
| `NO SUPPLIERS FOUND` | rule §4B | screenshot to confirm the panel, note `CANT ROUTE ORDER CONTACT GIDEON`, then `node gideon.js --add <ORDER#> --send-if-new` **once at the end of the run** |
| `SHARED STOCK NOT OFFERED - CONTACT GIDEON` | the dialog offered nothing or only barred vendors, but the "In shared warehouses" modal shows a real warehouse with stock on hand (listed in the refusal) | rule §4B: email Gideon, note `CANT ROUTE ORDER CONTACT GIDEON`. **Not** a barred-only stop and **not** rule §4A — RP has the part and is failing to offer it (Adam, 09-15, `02-15186-86132`). |
| `TONSA ONLY - STOP AND TELL ADAM` | §5 Tonsa block | **Stop.** Do not route, and do **not** hand to stock-sync-guard — the part is in stock, RP's price for Tonsa is simply untrustworthy. Note `CANT ROUTE - TONSA BLOCKED (RP PRICING) - NEEDS ADAM`, alert via `alert.js`. |
| `TONSA - BLOCKED WHILE RP PRICING IS WRONG` | Tonsa was the named pick | Pick a different supplier. |
| `GRIFFIN ONLY - STOP AND TELL ADAM` | §5 Griffin rule | **Stop.** Do not route, and do **not** hand to stock-sync-guard — no quantity change, no buyer message. Note `CANT ROUTE - GRIFFIN ONLY - ADAM NOTIFIED` and alert him with `alert.js` (**not** push — it no-ops unattended). He handles it. |
| `GRIFFIN + TONSA ONLY - STOP AND TELL ADAM` | both bans fired | Same as above. |
| `GRIFFIN - NEVER ROUTE (Adam 09-08)` | Griffin was the pick | Re-run naming the best non-Griffin supplier. |
| `CAPITAL CHEVROLET ONLY - STOP AND TELL ADAM` | §5 Capital Chevrolet ban | **Stop.** Do not route, do **not** hand to stock-sync-guard — the part is in stock. Note `CANT ROUTE - CAPITAL CHEVROLET BARRED - ADAM NOTIFIED` and alert via `alert.js`. |
| `COCHRAN ONLY - STOP AND TELL ADAM` / `COCHRAN - NEVER ROUTE (Adam 09-17)` | §5 Cochran ban (re-barred 09-17) | **Stop.** Do not route, do **not** hand to stock-sync-guard. The shared-warehouse check runs first and turns it into `SHARED STOCK NOT OFFERED - CONTACT GIDEON` when another warehouse has the part; otherwise note `CANT ROUTE - COCHRAN ONLY - ADAM NOTIFIED` and alert via `alert.js`. |
| `CAPITAL CHEVROLET - NEVER ROUTE (Adam 09-10)` | Capital Chevrolet was the pick | Re-run naming the best remaining supplier. Capital **Ford** of Raleigh is a different dealer and is still routable. |
| `NO MATCHING SUPPLIER` | the named supplier was not offered | re-read the offered rows and pick again |

**Never apply a rule A action to a rule B order** (`ROUTING_RULES.md` §4B). The part IS in
stock; it is an RP back-end failure only they can fix. No quantity change, no buyer message,
no cancellation.

Before committing, cross-check profit (§4C): RP `Net Profit (Less actual shipping cost)`
against the eBay order detail. eBay's "Order earnings" does **not** subtract part cost, so it
looks healthy on a losing order — trust the RP figure. Negative or very thin → do not route,
note `CHECK PROFIT PRICING MAYBE OFF` with the numbers appended, e.g.
` - COST $194.70 VS $203.49 SALE - NET -$12.94`.

## Step 5 — Commit the route, or apply the exception rule

**Confirmation gate.** Committing spends money and is not reversible from here. Show the
operator the dry-run output — suppliers offered, the pick, cost, miles, ship-to — and wait for
an explicit go-ahead. This holds even when the pick is obvious. The only exception is a
standing instruction in this session naming a dollar cap, and then only under that cap.

```bash
node S:/CLAUDE/automation/route.js <RECORD#> --commit [supplierName]
```

Pass `supplierName` only to override the computed pick (e.g. skipping Cochran). The script
re-reads the selected row before submitting and verifies afterwards.

**Nothing to route (0 in stock and 0 in shared), and not an always-Vintage brand** → rule §4A.
**Hand the order to the `stock-sync-guard` skill and move on to the next one.** That skill owns
rule §4A end to end, including the buyer message and its approval gate — do not run `oos.js`
from here, and never message a buyer from this skill.

Pass it the order number, the RP record id, and the evidence that both stock figures read 0.
Come back and finish the rest of the queue afterwards.

**Never cancel the order yourself.** A seller cancellation puts a defect on Adam's record; a
buyer-requested one does not. This applies only to orders routing from 2026-09-02 onward.

A routed order whose supplier now reads "Out of Stock" is **normal** — it means we bought the
last one. Do not flag, re-route, cancel, or message the buyer.

## Step 5b — Verify exactly one active supplier order

`ROUTING_RULES.md` §1 — the rule that costs money. Immediately after routing, **reload the RP
order** and confirm the Supplier Orders panel shows exactly ONE non-Cancelled row, with the
supplier you intended. Then re-verify every PO created during the run again at the END of the
run: duplicates can appear a minute after the fact.

To fix a duplicate: Cancel Order on the NEWEST row, reason "Customer Requested Cancellation"
("Other" fails silently), reload, confirm it reads Cancelled. Cancelling an RP purchase order
is permitted *only* to undo a duplicate route.

## Step 5c — Re-verify every MESSAGED TO CANCEL listing is still at zero

```bash
node S:/CLAUDE/automation/oos-audit.js --fix
```

Quantity can climb back after rule §4A zeroes it (`09-15158-00641` went 0 → 1 available
overnight on 09-12 with no stock anywhere — most likely RP's listing sync). This re-reads every
`MESSAGED TO CANCEL` listing and re-zeroes any that can still sell, logging to `rule-a.log`.
Report re-zeroes by order ID; anything still above 0 after the fix goes to Adam. Details in
`stock-sync-guard` Step 4b.

## Step 6 — Record it

Three records, in this order, immediately after the route is verified — before moving to the
next order, so an interrupted run never double-orders.

**1. The eBay note.** Every order touched gets one, and on a **multi-item order every line
item** gets it (there is no order-level note menu on those).

```bash
node S:/CLAUDE/automation/note.js <ORDER#> "ROUTED TO <SUPPLIER> PO <n>"
```

Use the exact wording from `ROUTING_RULES.md` §6 — `ROUTED TO <SUPPLIER> PO <n>`,
`SENT TO AMANDA`, `OUT OF STOCK AT VINTAGE - ADAM`, `MESSAGED TO CANCEL MM-DD`,
`CANT ROUTE ORDER CONTACT GIDEON`, `CHECK PROFIT PRICING MAYBE OFF`,
`CARVANA ORDER PLEASE CANCEL`, `POSSIBLE CARVANA - ADAM CHECK BUYER`. The note is what
stops the next run re-picking the order, so it is never optional.

**Write the note text plain.** `ROUTING_RULES.md` renders some of these in `**bold**`;
those asterisks are markdown emphasis in that document, not part of the note. Live notes
read `CANT ROUTE ORDER CONTACT GIDEON`, never `**CANT ROUTE ORDER CONTACT GIDEON**`.

**2. Local cache** — append to `.claude/state/routed-orders.json`:

```json
{
  "orderId": "12-34567-89012",
  "sku": "ABC-123",
  "quantity": 1,
  "supplier": "…",
  "rpRecordId": "…",
  "supplierOrderId": "…",
  "unitCost": "…",
  "status": "routed",
  "routedAt": "2026-09-08T14:03:00Z"
}
```

Use `status: "oos-messaged"` for rule §4A orders, `"gideon"` for §4B, `"carvana"` for §4D
holds, and `"manual"` for anything else flagged for Adam.

**3. `S:\CLAUDE\ROUTING_STATE.md`** — update at the end of every run (`ROUTING_RULES.md` §8).

Do not push tracking back to eBay in this skill. Revolution Parts ships and supplies tracking
on its own schedule; uploading it is a separate step.

## Hard rules

- **Never place the same Revolution Parts order twice.** RevolutionParts itself is the
  guard — an existing supplier order or a `Routed supplier order` activity entry means the
  order is done. The state file is a fast local cache, not the source of truth, and a blank
  Warehouse field means nothing. Check both (Step 1a and 1b) at the start of every run and
  write to the state file the moment an order lands.
- **Never re-route to the warehouse that just cancelled** (rule §4E). It cancelled because
  it cannot fill the part; sending it straight back earns a second cancellation and loses
  another day. Name the next best supplier explicitly instead.
- **Never pick a warehouse `routing-rules.md` does not cover.** Ask instead.
- **Never send the buyer a message from this skill** — that is stock-sync-guard's job, with
  its own gate.
- If Revolution Parts shows a login screen, stop and tell the operator to log in. Do not
  attempt to enter credentials.

## Finishing

Summarize the run: routed (with warehouse and cost), handed to stock-sync-guard, flagged
manual, skipped as already-routed. Name any order that needs the operator to do something.
