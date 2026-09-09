---
name: stock-sync-guard
description: Handle the eBay/Revolution Parts sync lag — a part sold on eBay that Revolution Parts cannot fill. Implements rule §4A of ROUTING_RULES.md via oos.js — zeroes the eBay quantity, messages the buyer to request cancellation, notes the order, and tracks the request through to a refund. Use when a part is out of stock on Revolution but still live on eBay, when route-orders hands off an unfillable order, or when asked to sweep for stale quantities.
---

# Out-of-stock sync guard — rule §4A

eBay quantity and Revolution Parts stock drift apart. A part sells on eBay after Revolution
has gone to zero, and the order cannot be filled.

**This skill is the executor for rule §4A of `S:\CLAUDE\ROUTING_RULES.md`.** That file is the
policy and the final word; this one is the running order. `route-orders` detects the condition
and hands off here — it never messages a buyer itself.

Two ways in: `route-orders` hands off an order it could not route, or the operator runs this
directly on an order they already know is dead.

## Do NOT run this skill when

Every one of these looks like an out-of-stock order and is not. Check them before anything.

| Condition | Why not | Do instead |
| --------- | ------- | ---------- |
| **Rule §4B** — stock shows in shared warehouses but routing returned `NO SUPPLIERS FOUND` | The part IS in stock; it is an RP back-end failure only they can fix. Zeroing the listing throws away a fillable sale. | Note `CANT ROUTE ORDER CONTACT GIDEON`, queue with `gideon.js`. **No rule A action at all.** |
| **The BUYER requested the cancellation** (rule §4F) | The part is not out of stock, and the buyer has already asked — messaging them to ask is nonsense, and zeroing the listing stops it selling to everyone else. | Note `BUYER REQUESTED CANCEL - ADAM TO APPROVE` and report it. Adam approves and refunds. |
| **Griffin Auto Group is the only supplier** | Adam handles these himself. Griffin has cancelled every order since we started with them, but the part is not out of stock. | Stop. Alert Adam with `alert.js` (SMTP — push no-ops unattended), note `CANT ROUTE - GRIFFIN ONLY - ADAM NOTIFIED`. **No quantity change, no buyer message.** |
| **Always-Vintage brand** at 0/0 | 0/0 is the NORMAL presentation for these — RP holds no catalog data. Worth $759.56 on order 07-15119-33809 (2026-09-02). | `python S:/CLAUDE/automation/inv.py "<SKU>"`. In the file → `vintage-batch.md`. Not in the file → note `OUT OF STOCK AT VINTAGE - ADAM` and stop; Adam handles it. |
| **Already-routed order whose supplier now reads "Out of Stock"** | Normal — it means we bought the last one. (Claude raised 12 false positives on 09-02/09-03; Adam corrected it.) | Nothing. Do not flag, re-route, cancel, or message. |
| **Order already carries an eBay note** | It has been handled by an earlier run. The note is also what scopes this rule to orders from 2026-09-02 onward. | Skip it. |
| **An active (non-Cancelled) supplier order exists** | The order is done. | Leave it alone whatever the stock column says now. |
| **Green `N In stock` > 0 at Vintage or Pinnacle** | We own it. | `vintage-batch.md`, or the Pinnacle hold. |

Routing is always attempted **first**. This rule only fires on an order that could not be
routed because no stock existed at the moment we tried. Never message a buyer about
cancelling before a route has been attempted — that kills an order we could have filled.

## Step 1 — Confirm there is genuinely nothing to route

On the RP order detail page (`node S:/CLAUDE/automation/rp.js <ORDER#>`), confirm the
line-item panel reads **`0 In stock` AND `0 In shared warehouses`**. Take a snapshot as
evidence.

Do not use the Warehouse column or the "Warehouse:" dropdown — both are meaningless labels.
Inventory is volatile; re-read it fresh rather than trusting a figure from an earlier run.

If any warehouse can still ship it, this is a routing decision, not a cancellation — send it
back to `route-orders`.

## Step 2 — Run the rule

`oos.js` performs all three actions in the required order and carries the guards. Prefer it
over doing any of this by hand.

```bash
node S:/CLAUDE/automation/oos.js <ORDER#>            # dry run — checks every guard
node S:/CLAUDE/automation/oos.js <ORDER#> --commit   # do it
```

It:

1. Sets the listing quantity to **0**. The listing stays alive, hidden from search, and
   returns when restocked — Out-of-Stock Control is enabled on the account. **Do not end the
   listing**; `ebay_end_listing` is denied in this project's settings for that reason.
2. Messages the buyer with `S:\CLAUDE\MESSAGED TO CANCEL TEMPLATE.txt` **verbatim**, subject
   `Regarding your eBay order <ORDER#>`, asking them to request the cancellation.
3. Notes the eBay order `MESSAGED TO CANCEL MM-DD` (today's date).

It refuses if the order already has a note, RP shows any stock, an active supplier order
exists, the brand is always-Vintage, or the eBay user token is missing. **If the buyer message
fails it does not write the note**, so the next run retries cleanly.

**Show the operator the dry-run output and wait for approval before `--commit`.** This sends a
message to a real customer under Adam's seller account.

Do not compose your own wording. The template is the approved text; a bespoke message is how
tone drifts and cases get opened.

## Why buyer-requested and not seller-cancelled

A seller cancelling for "out of stock" takes an eBay **defect**. A buyer-requested
cancellation does not. So: message the buyer → the buyer requests cancellation → Adam accepts
and refunds. Adam approves the request when it arrives, and sends a second message and cancels
the next day if the buyer has not acted — **which is what the date in the note is for.**

**Never relabel a stock-out as a buyer request.** The `BUYER_ASKED_CANCEL` reason code is only
truthful after the buyer has actually asked. Using it on an unanswered order to dodge the
defect misreports the cancellation to eBay. When the buyer does not respond, the honest
options are `OUT_OF_STOCK_OR_CANNOT_FULFILL` and its defect, or waiting — and that call
belongs to Adam, not to you.

This server has no seller-cancel tool; `ebay_get_cancellation_requests` is how you see a
request arrive. (See "Known gaps" in `.claude/README.md` for the Post-Order endpoints.)

## Step 3 — Log it

Update the order's entry in `.claude/state/routed-orders.json` (append one if absent):

```json
{
  "orderId": "12-34567-89012",
  "sku": "ABC-123",
  "status": "oos-messaged",
  "quantityZeroedAt": "2026-09-08T14:05:00Z",
  "buyerMessagedAt": "2026-09-08T14:06:00Z",
  "noteDate": "09-08",
  "cancelReceivedAt": null,
  "refundedAt": null
}
```

Report the order ID to Adam, and update `S:\CLAUDE\ROUTING_STATE.md` at end of run.

## Step 4 — Follow through

The job is not done when the message sends. On each later run of this skill or of
`route-orders`, call `ebay_get_cancellation_requests` and reconcile against every entry with
`status: "oos-messaged"`:

- **Request arrived** → tell Adam to accept and refund in Seller Hub, then set
  `cancelReceivedAt`.
- **No request by the day after `noteDate`** → surface it. Per §4A, Adam sends a second
  message and cancels the next day if the buyer has not acted. A silent buyer becomes an
  item-not-received case, which is worse than a defect. The call is Adam's.

## Standalone sweep

Run without a specific order to catch drift *before* it sells — this is additive to §4A, not
part of it:

1. `ebay_get_offers` (or `ebay_get_active_listings`) for listings with quantity above zero.
2. Spot-check the ones that matter most in Revolution Parts — highest-velocity SKUs first; a
   full catalog crawl is slow and worth scheduling separately.
3. Zero the quantity on anything Revolution shows at zero. **No buyer message and no eBay
   note** — nobody has bought it yet. That is the whole point of catching it here.

The always-Vintage exclusion still applies: those SKUs read 0/0 normally and must not be
zeroed on that basis.

## Hard rules

- **Never cancel an order as the seller.** Ask the buyer; escalate to Adam if they go quiet.
- **Never apply any rule A action to a rule §4B order.** No quantity change, no buyer message,
  no cancellation.
- **Never run this before a route has been attempted.**
- **Never send a buyer message without showing the dry run first**, and never with wording
  other than the approved template.
- **Never zero a quantity without confirming 0 in stock and 0 in shared yourself.** A misread
  table takes a live listing off the market.
- **Never issue a refund before the order is actually cancelled** — `ebay_issue_refund` is
  gated behind confirmation and is not part of the normal path.
