---
name: route-orders
description: Route new eBay orders to a Revolution Parts warehouse. Pulls unfulfilled eBay orders, looks each part up in Revolution Parts, picks the warehouse using routing-rules.md, places the order, and hands out-of-stock lines to stock-sync-guard. Use when asked to route orders, check for new orders, process the order queue, or run the order sweep.
---

# Route eBay orders to Revolution Parts

This is the Claude Code replacement for the Cowork auto-routing browser flow. One order
at a time, end to end, with the money-spending step gated behind confirmation.

## Before you start

Read these three files. Do not improvise around them.

- `.claude/skills/route-orders/routing-rules.md` — the warehouse selection rules. **This is
  the operator's rules, not yours.** If a section still reads `TODO`, stop and ask rather
  than guessing which warehouse to use.
- `.claude/skills/route-orders/revolution-parts.md` — how to navigate the Revolution Parts UI.
- `.claude/state/routed-orders.json` — every order already routed. Read it with
  `cat .claude/state/routed-orders.json`. If it does not exist, treat it as `[]`.

## Step 1 — Pull the unrouted queue

Call `ebay_get_orders` with:

```
filter: "orderfulfillmentstatus:{NOT_STARTED|IN_PROGRESS}"
limit: 50
```

Drop any order whose `orderId` already appears in `routed-orders.json` with
`status: "routed"`. Drop any order with a non-empty `cancelStatus.cancelState` other than
`NONE_REQUESTED` — a cancel is already in flight; report it, don't route it.

Report the remaining queue as a short table (order ID, buyer, SKU, qty, ship-to state,
order date) before doing anything else. If the queue is empty, say so and stop.

## Step 2 — Resolve the part

For each order line, collect from the order payload:

- `lineItems[].sku` and `lineItems[].legacyItemId`
- `lineItems[].quantity`
- `fulfillmentStartInstructions[].shippingStep.shipTo` — the full ship-to address

The SKU is the link to Revolution Parts. If the SKU is missing or does not look like a
part number, call `ebay_get_inventory_item` for it, then fall back to the listing title.
If you still cannot identify the part with confidence, flag the order for manual handling
and move to the next one. Never guess a part number.

## Step 3 — Check stock in Revolution Parts

Follow `revolution-parts.md` to search the part. Capture, per warehouse:

- quantity on hand
- unit cost
- estimated transit days to the buyer's ZIP

Take a `browser_snapshot` of the results before deciding — that snapshot is the evidence
for the routing decision, and it goes in the summary.

## Step 4 — Branch on availability

**Any warehouse has enough stock** → continue to Step 5.

**No warehouse has enough stock** → this is the eBay/Revolution sync-lag case. Stop routing
this order and invoke the `stock-sync-guard` skill for it. Come back and continue with the
rest of the queue afterwards.

**Partial stock** (some warehouses have it, none have the full quantity) → do not split the
order on your own. Report it and ask.

## Step 5 — Pick the warehouse and place the order

Apply `routing-rules.md` in the order the tie-breaks are written there. State the decision
in one line before acting:

> Order 12-34567-89012 · SKU ABC-123 · qty 1 → **[warehouse]** — [the rule that decided it]

Then place the order in Revolution Parts per `revolution-parts.md`.

**Confirmation gate.** Placing the order spends money and is not reversible from here.
Show the operator the full basket — part, quantity, warehouse, unit cost, total, ship-to —
and wait for an explicit go-ahead before the final submit click. This holds even when the
routing decision is obvious. The only exception is a standing instruction from the operator
in this session that names a dollar cap, and then only under that cap.

## Step 6 — Record it

Append one entry to `.claude/state/routed-orders.json` immediately after the Revolution
Parts confirmation page renders — before moving to the next order, so an interrupted run
never double-orders:

```json
{
  "orderId": "12-34567-89012",
  "sku": "ABC-123",
  "quantity": 1,
  "warehouse": "…",
  "rpOrderId": "…",
  "unitCost": "…",
  "status": "routed",
  "routedAt": "2026-09-08T14:03:00Z"
}
```

Use `status: "oos-cancel-requested"` for orders handed to stock-sync-guard, and
`status: "manual"` for anything you flagged for the operator.

Do not push tracking back to eBay in this skill. Revolution Parts ships and supplies
tracking on its own schedule; uploading it is a separate step.

## Hard rules

- **Never place the same Revolution Parts order twice.** The state file is the guard;
  read it at the start of every run and write to it the moment an order lands.
- **Never route an order with an open cancellation request.**
- **Never pick a warehouse `routing-rules.md` does not cover.** Ask instead.
- **Never send the buyer a message from this skill** — that is stock-sync-guard's job, with
  its own gate.
- If Revolution Parts shows a login screen, stop and tell the operator to log in. Do not
  attempt to enter credentials.

## Finishing

Summarize the run: routed (with warehouse and cost), handed to stock-sync-guard, flagged
manual, skipped as already-routed. Name any order that needs the operator to do something.
