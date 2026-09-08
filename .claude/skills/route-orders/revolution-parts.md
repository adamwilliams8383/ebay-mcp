# Revolution Parts navigation notes

**Fill this in from your own dashboard.** The `route-orders` skill drives the browser
through the Playwright MCP server, and it needs the real URLs and control labels — a
snapshot-and-hunt approach on every run is slow and misclicks.

## Session

Revolution Parts login is **manual and done once**. The Playwright MCP server keeps a
persistent browser profile, so after you log in yourself the session is reused on later runs.

If Claude hits a login screen it must stop and say so. It never enters credentials, and no
Revolution Parts password belongs in this repo or in `.claude/`.

## URLs

| Purpose            | URL  |
| ------------------ | ---- |
| Dashboard          | TODO |
| Part search        | TODO |
| Stock by warehouse | TODO |
| Cart / checkout    | TODO |
| Order history      | TODO |

## Part lookup

TODO — write the click path. For example:

1. Navigate to the part search URL.
2. Type the SKU into the field labeled `TODO` and press Enter.
3. The results table lists one row per warehouse. The columns that matter are `TODO`
   (on hand), `TODO` (unit cost), `TODO` (estimated transit).
4. "Backorder" / "special order" appears as `TODO` — this does **not** count as in stock.

## Placing the order

TODO — write the click path, and name the exact label of the final submit control so the
confirmation gate in the skill lands on the right button.

1. TODO
2. TODO
3. Final submit is the button labeled `TODO`. **Stop here for operator confirmation.**
4. The confirmation page shows the Revolution Parts order number at `TODO` — capture it for
   the state file.

## Ship-to

TODO — how the eBay buyer address gets onto the Revolution Parts order: drop-ship fields,
a saved address book entry, or manual entry. Note any field that Revolution Parts
reformats or rejects (apartment lines and 9-digit ZIPs are common offenders).

## Known UI quirks

TODO — anything that trips up automation: a modal that appears on first search, a table
that lazy-loads, a warehouse selector that resets after a page change.
