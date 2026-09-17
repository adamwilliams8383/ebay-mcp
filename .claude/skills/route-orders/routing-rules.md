# Warehouse routing rules — POINTER, not the policy

**The authoritative policy lives at `S:\CLAUDE\ROUTING_RULES.md`.** Adam owns and edits it
directly; it is read fresh on every run. Do not copy it here — a second copy will drift, and
this policy has already cost real money when it was got wrong (see §1 of that file: duplicate
scheduled tasks triple-routed order 04-15124-05948 on 2026-09-02, POs 1796/1797/1798).

## Read before routing anything

1. `S:\CLAUDE\ROUTING_RULES.md` — business policy. Sections that decide a route:
   - **§0** hard security rules — never enter credentials; stop on a login page.
   - **§1** *the one rule that costs money* — every part ends on exactly ONE active supplier
     order. Re-verify after each route, and again at end of run.
   - **§3** classification — read the LINE ITEM inventory panel, never the Warehouse column.
   - **§4** exception rules A–D (out of stock / can't route / thin profit / Carvana).
   - **§5** vendor selection — cheapest first, distance to buyer as the tiebreak.
   - **§6** the exact eBay note text for every outcome.
2. `S:\CLAUDE\ROUTING_TECHNIQUES.md` — UI mechanics for the same screens (owner: Claude).
3. `S:\CLAUDE\CARVANA WATCHLIST.txt` — read fresh per order; an unreadable watchlist means
   HOLD, never "no match".

## Vendor selection, in brief (§5 is authoritative)

Cheapest supplier first; distance to the buyer breaks the tie.

- ~$5 dearer but much closer to the buyer → take the closer one.
- Farther but ~$10+ cheaper → take the cheaper one.
- Equal price → closer wins.

Standing vendor constraints — all of these override cheapest-first:

| Vendor | Rule |
| ------ | ---- |
| #1 Cochran Automotive Group | **Never route.** Barred again 2026-09-17 after cancelling `15-15160-83253` on a part RP showed in stock. Cochran-only → shared-warehouse modal first (Gideon if another warehouse has it), else **stop**, note `CANT ROUTE - COCHRAN ONLY - ADAM NOTIFIED`, alert via `alert.js`. Not rule §4A. |
| Griffin Auto Group | **Never route.** Cancelled every order since we started with them. If Griffin is the ONLY supplier offered → **stop**, tell Adam with `alert.js` (SMTP — push no-ops on unattended runs), note `CANT ROUTE - GRIFFIN ONLY - ADAM NOTIFIED`, and do **not** run rule §4A on it — no quantity change, no buyer message. He handles these. |
| Tonsa Automotive | **Blocked 2026-09-09** while RP's quoted price for them is wrong — *no* Tonsa quote can be trusted, so cheapest-first is unusable on their rows. Tonsa-only → do not route, note `CANT ROUTE - TONSA BLOCKED (RP PRICING) - NEEDS ADAM`, alert via `alert.js`. **Not** rule §4A — the part is in stock. Lift only when Adam says RP fixed it. |
| Tonkin Parts Center | A different vendor from Tonsa. Fine to route. |
| Capital Chevrolet of Wake Forest | **Never route.** Barred by Adam 2026-09-10. Capital-Chevy-only → **stop**, note `CANT ROUTE - CAPITAL CHEVROLET BARRED - ADAM NOTIFIED`, alert via `alert.js`, and do **not** run rule §4A on it. |
| Capital Ford of Raleigh | **A different dealer from Capital Chevrolet — not barred.** Ordinary supplier; routed to on 09-08 (PO 1448). The ban matches CAPITAL + CHEV, never `capital` alone. |
| Pinnacle Parts | On hold entirely. Do not route; note and flag. |

## Where stock actually comes from (§3)

Read the line-item inventory panel on the RP order detail page:

- green `N In stock` > 0 → **we own it.** Only Vintage and Pinnacle hold own stock. Open the
  "Your warehouses" modal: *Vintage Direct Upload, Beaver Dam WI* → Vintage order (check the
  day's inventory file first); *Pinnacle Parts, Coral Springs FL* → on hold.
- green `0 In stock` + blue `N In shared warehouses` > 0 → **route via dropship.**
- `0 In stock` and `0 In shared warehouses` → exception rule §4A.

**The Warehouse column in the order list and the "Warehouse:" dropdown on the detail page
both mean nothing.** They are default labels; a fully routed order still shows a blank or
"Not Assigned" warehouse. Verified independently on order 74870144 (eBay 24-15129-48762),
routed to supplier order 1169 with an empty Warehouse column.

## Existing tooling this skill must not duplicate

`S:\CLAUDE\automation` already implements much of this flow. Prefer these over reimplementing:

| Tool | Purpose |
| ---- | ------- |
| `route.js` | drives the RP dropship dialog |
| `oos.js <ORDER#> [--commit]` | rule §4A — quantity to 0, buyer message, eBay note; refuses on every guard |
| `gideon.js --add <ORDER#> --send-if-new` | rule §4B — batches can't-route orders to Gideon Williams at RP |
| `inv.py "<SKU>"` | Vintage inventory-file lookup by full eBay Custom Label |

Live per-order state is tracked in `S:\CLAUDE\ROUTING_STATE.md`, updated at the end of
every run.
