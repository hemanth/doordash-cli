---
name: dd-office-supplies-concierge
description: >-
  Run a bot that accumulates ad-hoc office supply / snack restock requests
  from employees (e.g. "@DoorDash-bot we're out of printer paper") into one
  running cart all week using dd-cli / the DoorDash MCP, then sends the
  final cart to an approver for review at a fixed weekly time before
  checking out.
---

# Office supplies & snacks concierge

Use this for a Slack-bot pattern where requests trickle in all week and get
batched into one order, checked out only after a human approves:

> "Whenever employees tag @DoorDash-bot, add their requested items to the
> cart. Send the final cart to me every Friday at 3pm. I'll review it and
> confirm you're good to place the order."
>
> - "@DoorDash-bot we're running low on diet coke, can you add a 12 pack?
>   Oh also get more popchips."
> - "@DoorDash-bot can you order the super large sticky notes for tomorrow's
>   brainstorm?"
> - "@DoorDash-bot we're out of printer paper. Get us 6 packs."

Unlike `dd-group-cart-slackbot`, this is normally a **single cart owned by
one host identity** (the bot, or whoever's card is on file for office
spend) — employees don't need their own sub-cart or guest identity, since
nobody is picking their own individual meal; they're just flagging a
restock need for a shared cart. If your org wants per-requester spend
tracking on this same cart, the guest mechanism from
`dd-group-cart-slackbot` still applies — treat each requester as a guest so
their items are traceable in the cart, and read that skill for the
`--guest-json` mechanics.

## This is not a restaurant search

Office supplies, snacks, and drinks are retail/grocery verticals, not
restaurants — `dd-cli search` is restaurant-only and will usually return
zero results for "printer paper" or "diet coke 12 pack." Use:

```
dd-cli find-nearby-stores --vertical retail \
  --address-id <office_address_id> --intent "..."
# or --vertical grocery / convenience for snacks and drinks
dd-cli find-items --store-id <id> --query "<item>" --intent "..."
```

Cache the store id(s) your office actually uses (e.g. one office-supply
retailer, one grocery/convenience store for snacks/drinks) so a routine tag
doesn't re-search from scratch every time.

## Per-request flow (runs continuously through the week)

### 1. Parse the tagged message into a store + item + quantity

- "12 pack" / "6 packs" / "super large" are quantity/variant signals — map
  them to the specific SKU's `find-items` result rather than the generic
  product name where more than one size/pack exists. If `find-items`
  returns multiple plausible matches (e.g. several diet coke pack sizes),
  ask a one-line clarifying question in-thread before adding, rather than
  guessing.
- A single tag can contain multiple asks ("low on diet coke... also get
  more popchips") — split it into separate line items.

### 2. Check for the week's running cart

```
dd-cli cart list --store-id <id> --intent "..."
```

If a cart already exists at this store from earlier in the week, keep
using its `cart_uuid`. Persist `{store_id: cart_uuid}` in your bot's own
state so every request through the week lands in the same cart instead of
creating a new one per tag.

### 3. Add the item

```
dd-cli item-details --store-id <id> --item-id <id> --intent "..."
dd-cli cart add-items --store-id <id> --menu-id <id> \
  --items-json '[{"item_id":"<id>","item_name":"Diet Coke 12-Pack","quantity":1}]' \
  --cart-uuid <existing_cart_uuid_if_any> --intent "..."
```

`--menu-id` for retail/grocery comes from `item-details`'s `menu_id` field
(or from `build-grocery-list`'s top-level `menu_id` if you batch multiple
requested items through that command instead — useful when several tags
land close together and you want one multi-item add).

React to the Slack message (e.g. ✅) once the add succeeds so the requester
knows it landed, and surface a plain failure message if `item_errors[]`
comes back non-empty (e.g. item unavailable) instead of silently dropping
it.

### 4. Multiple stores in one week

If requests span more than one vertical (office supplies + snacks/drinks
from different retailers), keep a separate running cart per store — a
consumer/bot account can only have one open cart per store, and `order
preview`/`order submit` operate on one `cart_uuid` at a time. You'll run
the review-and-checkout step (below) once per store cart.

## Weekly review + checkout (at the stated fixed time)

### 1. Price every open cart from the week

```
dd-cli cart show --cart-uuid <id> --intent "..."
dd-cli order preview --cart-uuid <id> --intent "..."
```

### 2. Send the approver the full picture, then wait for their explicit go-ahead

Post each cart's contents + price to the approver (e.g. via DM) at the
stated time. Do **not** proceed to submit until they explicitly confirm —
"I'll review it and confirm you're good to place" is the approver
explicitly asking for a human-in-the-loop gate here, not a formality to
skip for convenience. If they ask for a change (remove an item, swap a
size), use `cart remove-item` / `cart add-items` and re-preview before
asking again.

### 3. On explicit approval, submit each cart

```
dd-cli order submit --cart-uuid <id> --tip-cents <n> --yes --intent "..."
dd-cli order status --order-uuid <id> --intent "..."
```

Pickup or delivery-with-no-dasher-tip situations still need
`--tip-cents 0` passed explicitly rather than omitted-and-assumed — see
`order submit --help` for the current default.

### 4. Reset for next week

After a submitted cart, treat its `cart_uuid` as spent — clear your bot's
`{store_id: cart_uuid}` mapping for that store so next week's first tagged
request creates a fresh cart rather than trying to reuse the submitted one.

## Common pitfalls

- **Don't let a `find-items` ambiguity silently resolve to the wrong SKU.**
  A "12 pack" request landing on a single-can result is exactly the kind of
  error nobody notices until the delivery arrives wrong.
- **Don't checkout without the weekly human approval**, even if the cart
  looks routine — that step exists specifically so someone reviews spend
  before it's charged.
- **Track requester attribution if the approver wants to know who asked for
  what.** Plain `cart add-items` doesn't tag a line item by requester; if
  that matters, use the guest mechanism from `dd-group-cart-slackbot` (one
  guest identity per requester on this same cart) instead of one anonymous
  host-only cart.
