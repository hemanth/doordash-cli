---
name: dd-team-recurring-order
description: >-
  Set up a recurring order placed on behalf of a whole team by one person
  (e.g. Friday all-hands treats, Mon-Wed team lunch) using dd-cli / the
  DoorDash MCP — one cart, sized by headcount from a calendar invite,
  respecting dietary constraints and a per-person or total budget. This is a
  single-cart order the organizer builds and pays for, not a shared cart
  employees add to themselves — for that, see dd-group-cart-slackbot.
---

# Recurring team order (organizer-built, single cart)

Use this when one person sets up a standing order **on behalf of** a group,
and that person (or the automation running as them) picks the items and
pays — nobody else needs to add their own items to the cart. Two examples
from the DoorDash MCP 1-pager:

> "Order treats for the all-hands every Friday. Rotate between sweets,
> specialty drinks, and savory snacks. Check the calendar invite to see how
> many people accepted the invite at 12pm the day before to order the right
> quantity. Deliver 30 minutes before all-hands starts. Budget is $250
> including fees and tip."

> "Order lunch for the team every Monday-Wednesday, arriving at 12pm. Rotate
> through our usual spots and try new places too. We have 3 vegetarians and
> 2 gluten-free. Budget is $25 per person."

If instead you want employees to add their **own** items to a shared cart
(so each person picks what they personally want), that's a group cart — use
`dd-group-cart-slackbot` instead. This skill is for a single organizer
deciding the whole order.

## Prerequisites

- The organizer (or the automation's service identity) has run
  `dd-cli login` / has `DD_CLI_ACCESS_TOKEN` set for headless runs.
- A calendar tool/skill is available if headcount comes from an invite (not
  part of dd-cli — use whatever calendar integration your agent has).
- Know the team's fixed constraints once, up front, and re-apply them every
  run: dietary restrictions (vegetarian/gluten-free counts), delivery
  address, budget shape (per-person vs. total-with-fees).

## Per-run flow

### 1. Determine headcount

Pull the calendar invite's accepted-attendee count at the stated check-in
time (e.g. "at 12pm the day before"). This drives both quantity and, for a
fees-inclusive total budget, the per-item price ceiling.

### 2. Pick stores/items for this run, honoring the rotation and constraints

- Keep a running memory (in whatever state your automation persists between
  runs — not part of dd-cli) of which categories/spots were used recently so
  the rotation doesn't repeat back-to-back. The prompt examples call for
  category rotation (sweets → drinks → savory) or spot rotation (usual +
  occasional new).
- Discover new spots: `dd-cli search -q "<category>" --address-id
  <office_address_id> --intent "..."`.
- Guarantee dietary coverage explicitly — don't just hope the assortment
  covers it. If 3 people need vegetarian, confirm at least 3 vegetarian-
  appropriate line items are in the cart (check `menu` /
  `restaurant-item-details` item tags/descriptions, or ask the organizer once
  which dishes at a rotation spot qualify and remember that for reuse).
- For a snack/drink assortment ("rotate between sweets, drinks, savory"),
  build a multi-item cart at one store per category via repeated
  `cart add-items` calls, or use `find-nearby-stores` /
  `find-items` for retail-style snack/beverage stores.

### 3. Build one cart, sized to headcount

```
dd-cli cart add-items --store-id <id> --menu-id <id> \
  --items-json '[{"item_id":"<id>","item_name":"<dish>","quantity":<headcount_or_platter_count>}]' \
  --intent "..."
```

For a platter/tray item, quantity is usually the tray count, not the
headcount directly — check the item's serving size in `menu` /
`item-details` before assuming 1 tray = 1 person.

### 4. Preview against the budget

```
dd-cli order preview --cart-uuid <id> --intent "..."
```

- **Total-with-fees-and-tip budget** (e.g. "$250 including fees and tip"):
  sum subtotal + fees + a reasonable tip and confirm it's under cap before
  moving on. If it's close to the ceiling, price the tip in — don't leave it
  as an afterthought that blows the budget at submit time.
- **Per-person budget** (e.g. "$25 per person"): divide the previewed total
  (excluding or including tip, per what was actually stated) by headcount
  and confirm it's under the per-person cap.
- If either check fails, drop an item or swap to a cheaper store/dish and
  re-preview. Don't submit over budget.
- If a work/company budget applies to the whole team order, pass
  `--include-work-benefits` and pick the right budget id — see
  `dd-lunch-autopilot` for the exact field paths
  (`quote.expense_order_options...`).

### 5. Time the delivery

```
dd-cli order preview --cart-uuid <id> --scheduled-time <UTC ISO8601> --intent "..."
```

Compute the scheduled time from the stated offset (e.g. "30 minutes before
all-hands starts", or a fixed "arrives at 12pm") and the calendar event's
actual start time when relevant. Pass the identical `--scheduled-time` to
`order submit`.

### 6. Submit and confirm

```
dd-cli order submit --cart-uuid <id> --tip-cents <n> --scheduled-time <ts> --yes --intent "..."
dd-cli order status --order-uuid <id> --intent "..."
```

Because this is a recurring unattended job, build in a lightweight approval
step analogous to `dd-lunch-autopilot` step 4 if the organizer wants to
review before it's charged (e.g. post the priced cart to their DM by a
cutoff time); if the organizer explicitly wants full autopilot with no daily
review, that's a valid choice too — just make sure it was their explicit
choice, not an assumption.

## Common pitfalls

- **Headcount lag**: if the calendar check-in time (e.g. noon the day
  before) is earlier than when RSVPs typically finalize, headcount may
  undercount late accepters. Consider a second, closer-to-delivery
  headcount check if the organizer's instructions allow re-sizing, or pad
  slightly and say so when reporting the order back.
- **Dietary coverage is a hard requirement, not a rotation nice-to-have** —
  verify it explicitly every run, even when the rotation naturally tends to
  include some kind of vegetarian/gluten-free option.
- **One cart, one open-cart-per-store rule still applies.** Run `cart list
  --store-id <id>` before creating a new cart each run in case a previous
  run's cart didn't get submitted or cleaned up.
