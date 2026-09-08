---
name: stock-sync-guard
description: Handle the eBay/Revolution Parts sync lag — a part sold on eBay that Revolution Parts shows out of stock. Messages the buyer to request cancellation, zeroes the eBay quantity so it stops selling, and tracks the cancel request through to a refund. Use when a part is out of stock on Revolution but still live on eBay, when route-orders hands off an unfillable order, or when asked to sweep for stale quantities.
---

# Out-of-stock sync guard

eBay quantity and Revolution Parts stock drift apart. A part sells on eBay after Revolution
has already gone to zero, and the order cannot be filled. This skill handles that case:
ask the buyer to cancel, stop the listing from selling again, and follow the cancel through.

Two ways in: `route-orders` hands off an order it could not fill, or the operator runs this
directly on an order they already know is dead.

## Why buyer-requested and not seller-cancelled

A seller cancelling for "out of stock" takes an eBay defect. A buyer-requested cancellation
does not. So the sequence is: message the buyer → the buyer requests cancellation → you
accept it and refund. That is why this skill sends a message rather than cancelling
outright, and why the eBay MCP server has no seller-cancel tool to reach for.

The actual accept-and-refund happens in eBay Seller Hub once the buyer's request lands.
`ebay_get_cancellation_requests` is how you see it arrive; this server has no seller-cancel
tool (see "Known gaps" in `.claude/README.md` for the Post-Order endpoints that would add one).

**Never relabel a stock-out as a buyer request.** If the loop is ever automated, the
`BUYER_ASKED_CANCEL` reason code is only truthful after the buyer has actually asked. Using
it on an unanswered order to avoid the defect misreports the cancellation to eBay. When the
buyer does not respond, the honest options are `OUT_OF_STOCK_OR_CANNOT_FULFILL` and its
defect, or waiting — and that call belongs to the operator, not to you.

## Step 1 — Confirm it is really out of stock

Before messaging anyone, verify in Revolution Parts that **no** warehouse can fill it. Check
every warehouse in `.claude/skills/route-orders/routing-rules.md`, not just the preferred
one. Take a `browser_snapshot` as evidence.

A part that one warehouse can still ship is a routing decision, not a cancellation — send it
back to `route-orders`.

## Step 2 — Stop the bleeding on eBay

Do this before the buyer message. Every minute the listing stays live is another order you
will have to cancel.

Set the offer quantity to zero with `ebay_bulk_update_price_quantity`, keyed on the SKU from
the order line. If the SKU maps to multiple offers, zero all of them.

Zeroing quantity ends the listing's availability without ending the listing, so it can be
restocked later. Do **not** use `ebay_end_listing` — it is denied in this project's settings
for exactly this reason.

Then re-read the offer to confirm the quantity actually took.

## Step 3 — Message the buyer

Draft with `ebay_send_message`:

- `otherPartyUsername` — the buyer, from the order payload
- `messageText` — the cancellation request, under 2000 characters
- `reference` — `{ referenceType: "LISTING", referenceId: <legacyItemId> }` so the message
  threads against the right listing

**Show the operator the draft and wait for approval before sending.** This message goes to a
real customer under the operator's seller account.

The message must:

- Apologize plainly and say the part is unavailable — a supplier stock issue, no excuses
  that blame the buyer or the platform.
- Ask the buyer to submit a cancellation request from their eBay order page, and say why
  (it is the fastest route to their refund).
- Promise the refund immediately on receipt of the request.
- Give a real alternative if one exists — an equivalent part in stock, or a restock date
  Revolution Parts actually shows. Do not invent a date.
- Never ask the buyer to close the case, leave positive feedback, or take the transaction
  off eBay.

Keep it short. Buyers reading a wall of text on their phone do not cancel; they open a case.

## Step 4 — Log it

Update the order's entry in `.claude/state/routed-orders.json` (append one if it is not
there yet):

```json
{
  "orderId": "12-34567-89012",
  "sku": "ABC-123",
  "status": "oos-cancel-requested",
  "quantityZeroedAt": "2026-09-08T14:05:00Z",
  "buyerMessagedAt": "2026-09-08T14:06:00Z",
  "cancelReceivedAt": null,
  "refundedAt": null
}
```

## Step 5 — Follow through

The job is not done when the message sends. On each later run of this skill or of
`route-orders`, call `ebay_get_cancellation_requests` and reconcile against every entry with
`status: "oos-cancel-requested"`:

- **Request arrived** → tell the operator to accept and refund it in Seller Hub, then set
  `cancelReceivedAt`.
- **No request after 48 hours** → surface it. A silent buyer becomes an item-not-received
  case, and that is worse than a defect. The operator decides whether to cancel from their
  side and take the hit.

## Standalone sweep

Run without a specific order to catch drift before it sells:

1. `ebay_get_offers` (or `ebay_get_active_listings`) for listings with quantity above zero.
2. Spot-check the ones that matter most in Revolution Parts — highest-velocity SKUs first,
   since a full catalog crawl through the browser is slow and worth scheduling separately.
3. Zero the quantity on anything Revolution shows at zero. No buyer message needed — nobody
   has bought it yet. That is the entire point of catching it here.

## Hard rules

- **Never cancel an order as the seller from this skill.** Ask the buyer; escalate to the
  operator if they do not respond.
- **Never send a buyer message without showing the draft first.**
- **Never zero a quantity without confirming Revolution stock yourself.** A misread table
  takes a live listing off the market.
- **Never issue a refund before the order is actually cancelled** — `ebay_issue_refund` is
  gated behind confirmation and is not part of the normal path here.
