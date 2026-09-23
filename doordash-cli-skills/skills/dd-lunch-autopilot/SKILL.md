---
name: dd-lunch-autopilot
description: >-
  Set up a recurring, personalized food order for one customer (e.g. "order
  my lunch every weekday") using dd-cli / the DoorDash MCP — rotating through
  past favorites, staying under a meal budget, previewing the pick before
  ordering, and falling back to a default choice if the customer doesn't
  respond in time.
---

# Lunch (or dinner) on autopilot

Use this skill when someone asks their agent to run their own recurring meal
order — e.g.:

> "Set up an automation to order lunch for me to the office every
> Monday-Thursday. Look at my past orders and rotate through my favorites,
> and throw in some new ideas of things I might like based on what I've
> ordered. Send me a preview of what you're going to order for me everyday
> by 11am on Slack. If I don't respond by 11:30am, order my go-to chicken
> caesar wrap as a backup option. Optimize the delivery time for when I have
> a free window between 12pm-1:30pm. Use my daily meal budget of $25."

This is a **personal** automation: one DoorDash identity, one recurring job
(a cron / scheduled agent run), no group cart involved.

## Prerequisites

- The customer has run `dd-cli login` once (or, for a scheduled/headless
  runner, exported a token via `dd-cli export-token` into
  `DD_CLI_ACCESS_TOKEN` on the machine that runs the automation).
- You (the agent) have a way to run on a schedule (cron, a scheduled agent
  run, a workflow trigger) — this skill is the *ordering logic* for each run,
  not the scheduler itself.

## Per-run flow

Each time the automation fires (e.g. weekday mornings):

### 1. Build the candidate list from history + one new idea

```
dd-cli order history --max 50 --days 90 --intent "..."
```

- Filter to restaurant orders (`fulfillment_type` /
  `order_target=ORDER_TARGET_RESTAURANT`) that were reorderable
  (`is_reorderable: true`).
- Pick from the rotation of past favorites. To keep it fresh, occasionally
  substitute a genuinely new pick: run `dd-cli search -q "<cuisine near
  favorites>" --address-id <office_address_id> --dashpass-only --intent "..."`
  and pick something plausible based on the customer 's past cuisine
  preferences — don't invent a random restaurant.
- Resolve the office delivery address once via
  `dd-cli --json-output address list --intent "..."` and reuse its
  `address_id` (label "Work" / "Office") for every search and order.

### 2. Build the cart

- For a repeat favorite: `dd-cli order reorder --order-uuid <past_order_uuid>
  --intent "..."` creates a new cart pre-filled with the same items — this is
  the fast path and preserves modifiers exactly.
- For a new pick: `dd-cli menu --store-id <id> --intent "..."` then
  `dd-cli cart add-items --store-id <id> --menu-id <id> --items-json '[...]'
  --intent "..."`.
- Always check for an existing open cart at that store first
  (`dd-cli cart list --store-id <id>`) so a retry of the same run doesn't
  double a cart.

### 3. Price it and check the budget

```
dd-cli order preview --cart-uuid <id> --intent "..."
```

- If a meal credit / company budget applies ("use my daily meal budget of
  $25" is exactly the work-benefits trigger phrase), pass
  `--include-work-benefits` on preview. Read
  `quote.expense_order_options.all_eligible_expense_order_budgets[]` and pick
  the one matching the stated budget; you'll pass its `id` as
  `--budget-id` on submit (with the matching `--team-id` from
  `quote.company_payment_info.team_order_info.team_id`).
- If the total (with fees/tip) would exceed the stated cap and there's no
  work-benefit budget covering it, drop back to a cheaper item on the same
  cart or swap to a cheaper favorite before previewing again. Don't submit
  over budget silently.
- Compute a tip. `order preview`'s `tips_suggestion_details` is the
  suggested-amount source when it's present.

### 4. Send the preview and wait for a response deadline

Post the picked item + price + restaurant to the customer's stated channel
(Slack, etc. — use whatever messaging skill/tool your agent has for that; not
covered by dd-cli) by the stated time (e.g. 11:00am). Hold the cart — do not
submit yet.

Wait until the stated deadline (e.g. 11:30am):

- **Customer approves / says nothing changes needed:** proceed to step 5
  with the previewed cart.
- **Customer requests a swap:** re-run steps 2-3 with the new pick.
- **No response by the deadline:** fall back to the stated default (e.g.
  "chicken caesar wrap"). Find it via `order history` (search past orders'
  `items[].name` for the fallback dish) and `order reorder` that order, or
  `cart add-items` it fresh if it's not in history. Preview it before
  submitting either way — never submit unpriced.

### 5. Time the delivery to the free calendar window

If a calendar tool is available, find the free window inside the stated
range (e.g. 12:00-1:30pm) and pass it as the scheduled delivery time:

```
dd-cli order preview --cart-uuid <id> --scheduled-time 2026-04-21T19:00:00Z --intent "..."
```

`--scheduled-time` needs a UTC-suffixed ISO 8601 timestamp — convert the
customer's local free-window start time to UTC first. Pass the **same**
`--scheduled-time` to `order submit` afterward; preview pricing is only valid
for the slot it was computed for.

### 6. Submit

```
dd-cli order submit --cart-uuid <id> --tip-cents <n> --scheduled-time <ts> \
  --budget-id <id> --team-id <id> --yes --intent "..."
```

(Omit `--budget-id`/`--team-id` for the no-work-benefits case; omit
`--scheduled-time` for ASAP.) `--yes` is required for a non-interactive/agent
run — without it, submit will hang waiting for a TTY confirmation.

Because this flow runs unattended, the human approval step is the Slack
preview/deadline in step 4, not an interactive prompt at submit time — don't
skip step 4 to "save time."

### 7. Confirm and report back

```
dd-cli order status --order-uuid <order_uuid> --intent "..."
```

Poll every ~15 seconds until placed, then report the confirmed order (store,
items, ETA) back to the customer's channel. Don't report success right after
`submit` returns — that only means the order was accepted into processing,
not that payment/placement finished.

## Common pitfalls

- **Don't re-run `order submit` on retry.** If a run fails partway and you
  retry, check `dd-cli order history` / `order status` first — submit has no
  idempotency and a duplicate call places a second, separately-charged
  order.
- **Respect the stated budget as a hard cap**, not a target — if the tip +
  fees push a $24 subtotal over $25, either lower the tip suggestion offered
  to the customer or pick a cheaper item; don't silently submit over cap.
- **The office address must resolve to a real saved address.** If
  `address list` has no "Work"/"Office" labeled entry, ask the customer once
  to save it (`dd-cli address add`) rather than guessing coordinates.
