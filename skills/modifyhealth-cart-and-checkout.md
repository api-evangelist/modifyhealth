---
name: Build a cart and complete a ModifyHealth checkout
description: Take a buyer from meal selection to a paid ModifyHealth order over UCP/MCP, with the idempotency key and buyer-approval rules the provider requires.
api: mcp/modifyhealth-mcp.yml
endpoint: https://modifyhealth.com/api/ucp/mcp
operations: [create_cart, update_cart, get_cart, create_checkout, update_checkout, get_checkout, complete_checkout]
generated: '2026-08-26'
method: generated
source: mcp/modifyhealth-mcp-tools.json (live tools/list, HTTP 200)
---

# Build a cart and complete a ModifyHealth checkout

This is a **real money** flow against a live storefront. ModifyHealth publishes no sandbox
or test mode — there is no way to rehearse it. Read the rules before the steps.

## Rules that bind the whole flow

- **Checkout requires contemporaneous human approval.** ModifyHealth states this in its
  own `agents.md`: agents must not complete payment without explicit buyer consent. If you
  cannot get approval at the moment of payment, do not call `complete_checkout`.
- **`complete_checkout` requires `meta.idempotency-key`.** It is a required field, not an
  optional one. Generate one key per logical purchase and reuse it verbatim on any retry.
  This is the only tool in the set that is idempotency-protected.
- **Cart and checkout mutations are NOT idempotent.** `create_cart`, `update_cart`,
  `create_checkout` and `update_checkout` declare no key. A blind retry after a timeout can
  duplicate state — call `get_cart` or `get_checkout` to reconcile instead of retrying.
- **Once the order exists, you cannot reverse it programmatically.** There is no refund
  tool. See the reversal section.
- **One shipping destination only.** The UCP profile declares
  `allows_multi_destination.shipping: false`. A cart cannot be split across addresses.
- **Continental US only, no P.O. boxes.** Free shipping; deliveries land on Friday.

## Steps

1. `create_cart` with `cart.line_items[]`, each `{quantity, item: {id: <variant id>}}`.
   Add `cart.buyer.email` and `cart.context.address_country` / `currency`.
2. `update_cart` to change quantities, set `cart.discounts.codes[]` (only if the buyer
   mentioned having a code — do not prompt for one), or attach fulfillment.
3. `create_checkout` to promote the cart to a purchasable state.
4. `update_checkout` to set the shipping destination and select a fulfillment method:
   `fulfillment.methods[]` with `type`, `destinations[]` and `selected_destination_id`.
5. `get_checkout` and read the totals back **to the buyer, in major units**. Get explicit
   approval.
6. `complete_checkout` with `meta.idempotency-key` and the payment object. The result
   carries the order id and a Thank You Page URL.
7. `get_order` to confirm.

## Reversal — know this before step 6

| Stage | How to reverse | Window |
|---|---|---|
| Cart open | `cancel_cart` | Any time before checkout completes |
| Checkout open | `cancel_checkout` | Any time before `complete_checkout` succeeds |
| Order placed | **No API path.** Phone 888-766-3439 | 7 days from delivery receipt, first order, discretionary |

Because the goods are perishable, ModifyHealth states it never accepts returns; refunds are
pro rata and case-by-case and may require photographic evidence. Multi-week programs
cancelled after that week's cancellation deadline still ship that week. Gift programs
cannot be cancelled and are nonrefundable.

**Practical consequence:** the last cheap moment to stop is `cancel_checkout`. Treat
`complete_checkout` as the point of no return and make the buyer's approval explicit there.

See `conventions/modifyhealth-conventions.yml` (reversibility) and
https://modifyhealth.com/policies/refund-policy.
