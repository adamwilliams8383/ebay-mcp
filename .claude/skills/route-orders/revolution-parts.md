# Revolution Parts navigation notes

Distilled from `S:\CLAUDE\ROUTING_TECHNIQUES.md` (§REVOLUTIONPARTS), which is the living
source — it gains notes every run. Read it when anything here does not match the screen.

## Session

Login is **manual and done once**. The Playwright MCP server keeps a persistent browser
profile, so the session is reused on later runs. Note this profile is separate from everyday
Chrome — logging in there does not carry over.

If Claude hits a login screen it must stop and say so. It never enters credentials, and no
RevolutionParts password belongs in this repo or in `.claude/`. (`ROUTING_RULES.md` §0.)

## URLs

Customer id is `22764`.

| Purpose            | URL |
| ------------------ | --- |
| Order search       | `https://manage.revolutionparts.com/customers/22764/orders?search=<ORDER#>` |
| Order detail       | `https://manage.revolutionparts.com/customers/22764/orders/<RECORD#>` |
| Open queue ("New") | `https://manage.revolutionparts.com/customers/22764/orders/open` |
| Supplier order     | `https://manage.revolutionparts.com/customers/22764/suppliers/<supplierId>/orders/<id>` |
| Purchases (buying) | `https://manage.revolutionparts.com/customers/22764/purchases/orders` |

**RP is a JavaScript SPA.** A server-side `fetch()` + DOMParser returns empty content — you
must navigate the tab to each page and read `document.body.innerText`.

## Order list

Tabs on the Open queue: **New** / Awaiting Payment / **In Progress** / Ready to Ship.
Columns: VIN, Order #, Placed, Ship By, Status, Store, Record #, Warehouse.
`Order #` is the eBay order number; `Record #` is the RP id used in the detail URL.

Cell indices for `[...tr.querySelectorAll('td')]`:

```
c[3]=order#   c[7]=status   c[9]=record#   c[10]=warehouse (meaningless)
c[15]=brand   c[18]=total
```

**Search sometimes returns an unrelated order** — always confirm `c[3]` matches the order you
searched for before using the row.

## Part lookup / stock

There is no part-search-and-compare screen. Stock is read off the **order detail** page:

- top badge — `Inventory` → `In Stock` | `Out of Stock` | `Unknown`
- line-item green — `/(\d+)\s+In stock/`
- line-item blue — `/(\d+)\s+In shared warehouses/`
- `Shipping: $X USD (included in price)`, `Your Cost: $X`, `Net Profit (Less actual shipping cost)`

Clicking the green `N In stock` link opens the **"Your warehouses"** modal; the first click
often only scrolls — re-locate and click again.

Inventory is **volatile** (order 05-15120-44280 showed 2 in shared on 09-01 and 0 on 09-02).
Always re-read fresh; never trust a number from an earlier run. `Your Cost` reading null on an
already-routed order is normal.

Classification rules for these signals live in `ROUTING_RULES.md` §3.

## Already routed?

Slice `innerText` from `Supplier Orders` to `Actions`; rows match `/^\d{3,5}\t/`; a row is
**active** unless it matches `/Cancel(l)?ed/`. The activity log entry reads
`Routed supplier order #… to supplier order #…`, and the line item reads
`Dropshipped to order <n>`.

Do **not** use the Warehouse column or the "Warehouse:" dropdown — both are meaningless
labels, and a routed order routinely shows "Not Assigned".

## Placing the order (dropship)

Control is **"Change Supplier"**, or **"ROUTE ORDER"** when there is no supplier panel yet.
It often needs TWO real clicks — confirm the modal opened by counting `input[type=radio]`.

1. Open the dialog; confirm it rendered by the radio count.
2. Select the supplier with a **real computer click**, never a JS click.
3. Read back **both** `r.checked` and the row's text to confirm the intended supplier — the
   Supplier Information panel can still show the previous one.
4. **Stop here for operator confirmation** (skill Step 5 gate).
5. Only then click **"Dropship Items"**.

**Never select the radio and click "Dropship Items" in the same JS call.**

If the dialog hangs on "Routing Order": REFRESH and check Supplier Orders **before** clicking
anything again. `NO SUPPLIERS FOUND` renders as a plain panel — screenshot before concluding
it, then apply `ROUTING_RULES.md` §4B (queue to Gideon).

On "problem processing your dropship": verify zero POs were created, then retry ONCE. Failing
twice → note `RP ERROR - <SUPPLIER> DROPSHIP FAILED TWICE - <PART> - NEEDS MANUAL ROUTE` and
flag Adam.

`S:\CLAUDE\automation\route.js` already drives this dialog — prefer it over hand-driving.

## After routing

Reload and confirm exactly ONE active supplier order (`ROUTING_RULES.md` §1). Capture the
supplier order number for the eBay note (`ROUTED TO <SUPPLIER> PO <n>`) and the state file.

## Known UI quirks

- Two clicks needed for the supplier modal and the "Your warehouses" modal.
- A supplier reading "Out of Stock" on an **already-routed** order is NORMAL — it means we
  bought the last one. Do not re-route, cancel, or flag it.
- Multi-item orders have no order-level note menu item; note every line item.
