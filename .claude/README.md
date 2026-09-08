# eBay → Revolution Parts order routing (Claude Code)

The Claude Code version of the Cowork auto-routing flow. Everything the agent needs lives in
this folder; the two skills are the entry points.

| Skill               | What it does                                                              |
| ------------------- | ------------------------------------------------------------------------- |
| `/route-orders`     | Pull unfulfilled eBay orders → look up the part → pick a warehouse → order |
| `/stock-sync-guard` | Part dead on Revolution but live on eBay → zero quantity, ask buyer to cancel |

## One-time setup

1. **Build the eBay MCP server** — `.mcp.json` points at `build/index.js`:

   ```bash
   npm install && npm run build
   ```

2. **Credentials** — copy `.env.example` to `.env` and run `npm run setup` for the OAuth
   flow. Verify with `npm run diagnose`.

3. **Start Claude Code in this directory** and approve both MCP servers when prompted
   (`ebay` and `playwright`). Check with `/mcp`.

4. **Log in to Revolution Parts once**, by hand, in the Playwright browser. The profile
   persists, so later runs reuse the session. Claude never types credentials.

5. **Fill in the two TODO files.** This is the part that actually matters:
   - `skills/route-orders/routing-rules.md` — your warehouse recommendations
   - `skills/route-orders/revolution-parts.md` — URLs and click paths

   Until they are filled in, `/route-orders` stops and asks instead of guessing a warehouse.
   That is deliberate: a wrong guess ships a real part to a real customer from the wrong place.

## Running it

```
/route-orders
```

For the "auto" part — Cowork watched the browser; Claude Code does not. Poll instead:

```
/loop 10m /route-orders
```

That re-runs the sweep every ten minutes for as long as the Claude Code session is open.
`.claude/state/routed-orders.json` is what stops a re-run from double-ordering, so never
delete it mid-day.

## What carries over from Cowork, and what does not

| Cowork                          | Here                                                          |
| ------------------------------- | ------------------------------------------------------------- |
| Fires on its own                | You start a session; `/loop` polls                            |
| Browser for both eBay and RP    | eBay via MCP API calls; browser only for Revolution Parts     |
| Routing rules held in the prompt| Version-controlled in `routing-rules.md`                      |
| Nothing remembered between runs | `state/routed-orders.json` prevents double-ordering           |

The eBay half being API-driven is the real gain: order pulls, quantity updates, and buyer
messages stop depending on eBay's DOM, which is what breaks browser automation most often.

## Safety model

`settings.json` splits tools three ways:

- **allow** — reads. Orders, listings, page snapshots. No prompt.
- **ask** — anything a customer or the market sees: placing an order, messaging a buyer,
  changing quantity or price, refunds.
- **deny** — irreversible listing destruction (`ebay_end_listing`, offer/item deletes) and
  reading `.env`.

To loosen a rule, move the tool between arrays. Do not add `mcp__ebay` wholesale to
`allow` — that grants all 299 tools, refunds and deletes included.

## Known gaps

- **No seller-cancel tool yet — but the API exists.** This server exposes
  `ebay_get_cancellation_requests` (read) only, so accepting a cancellation and refunding
  still happens in Seller Hub. The endpoints to close that loop are eBay's Post-Order API v2:
  `POST /post-order/v2/cancellation` (create, with a `cancelReason`) and
  `POST /post-order/v2/cancellation/{cancelId}/approve`. Neither is wrapped here yet —
  `src/api/order-management/orderIssueHelpers.ts` deliberately filters `getOrders`
  client-side instead. Wrapping them is a self-contained addition to `src/api/`.

  Reason codes decide whether you take a defect: `BUYER_ASKED_CANCEL` and `ADDRESS_ISSUES`
  do not, `OUT_OF_STOCK_OR_CANNOT_FULFILL` does. `BUYER_ASKED_CANCEL` is only accurate once
  the buyer has actually asked — which is why `stock-sync-guard` messages first and cancels
  second, and why it must never be used to relabel a stock-out.
- **Tracking upload is not automated.** Revolution Parts supplies tracking on its own
  schedule; pushing it back with `ebay_create_shipping_fulfillment` is a separate step.
- **Revolution Parts has no API here.** Everything on that side is browser-driven, so it
  breaks when their UI changes — that is what `revolution-parts.md` is for.
