# Warehouse routing rules

**Owned by the operator, not by Claude.** These are the recommendations the Cowork flow was
applying. Fill in every `TODO` below — until then the `route-orders` skill will stop and ask
rather than pick a warehouse on its own, which is the intended behavior.

Order matters: rule 1 is applied first, and later rules only break ties left by earlier ones.

## Warehouses

Fill one row per warehouse you route to.

| Code | Name | Location (city, ST) | Notes / restrictions |
| ---- | ---- | ------------------- | -------------------- |
| TODO | TODO | TODO                | TODO                 |
| TODO | TODO | TODO                | TODO                 |

## Rule 1 — Hard exclusions

Never route to a warehouse when any of these hold. Add or delete rows as needed.

- TODO — e.g. part category X is never sourced from warehouse Y
- TODO — e.g. warehouse Z does not ship to AK/HI/PR
- TODO — e.g. oversize/freight parts only from the warehouses that quote LTL

## Rule 2 — Stock

TODO. State the minimum. For example: the warehouse must show on-hand ≥ the order quantity;
"backorder" or "special order" status does not count as in stock.

## Rule 3 — Primary preference

TODO. This is the main recommendation the Cowork flow encoded. Pick the shape that matches
how you actually decide and delete the rest:

- **Closest to the buyer** — fewest transit days to the ship-to ZIP wins.
- **Cheapest landed** — lowest unit cost + inbound freight wins.
- **Fixed priority list** — always warehouse A if it has stock, else B, else C.
- **Split by part category** — TODO which categories go where.

## Rule 4 — Transit-time ceiling

TODO. Example: never route to a warehouse whose estimated transit exceeds the handling time
promised on the eBay listing; if only such a warehouse has stock, treat the order as an
exception and ask.

## Rule 5 — Tie-breaks

Applied in order when rules 1–4 leave more than one candidate.

1. TODO — e.g. lower unit cost
2. TODO — e.g. fewer transit days
3. TODO — e.g. better recent fill/damage record
4. TODO — e.g. the lower-numbered warehouse code, so the outcome is deterministic

## Exceptions that always come to the operator

Do not resolve these automatically:

- Order value over $TODO
- Quantity greater than TODO
- Freight / oversize parts
- Buyer address is a freight terminal, APO/FPO, or outside the lower 48
- The only in-stock warehouse violates a rule above

## Worked example

Once the rules above are filled in, keep one worked example here so the decision format
stays unambiguous:

> Order 12-34567-89012 · SKU ABC-123 · qty 1 · ship-to Dallas TX 75201
> Candidates: WH-EAST (2 on hand, $41.10, 4 days), WH-CENTRAL (5 on hand, $43.25, 2 days)
> Rule 3 (closest to buyer) → **WH-CENTRAL**. Cost tie-break not reached.
