---
name: dd-group-cart-slackbot
description: >-
  Run a shared DoorDash group cart from a bot account that employees add
  items to themselves — manually via a link, by replying in a Slack thread
  in natural language, or agentically from their own agent — then checks
  out on a deadline using dd-cli / the DoorDash MCP. Covers the guest
  sub-cart mechanism (--guest-json / guest_token) that lets the bot add
  items on behalf of people who never sign in to DoorDash themselves, and
  how it differs from a real participant joining with their own login.
---

# Group-cart ordering bot

This is the deepest skill in this pack — it documents the actual mechanics
of DoorDash's **group cart** feature via `dd-cli cart add-items`, because
every "shared team order" pattern (daily group lunch, all-hands sign-up
cart, a Slack bot employees @-mention) is a variation on it.

> "Every day, post a group cart link in #team-lunch from the
> @DoorDash-bot Slack account. Rotate through the spots we usually order
> from. If employees haven't added to the group cart by 11:30am, DM them a
> reminder. At 11:45am, place the order and post a confirmation. Ping the
> channel once it arrives."

## The two ways someone can be "in" a group cart

DoorDash's group cart has exactly two participant shapes. Get this
distinction right before building anything on top of it:

### 1. A real participant (has their own DoorDash login)

They join the group cart URL themselves (in a browser, or via their own
agent/dd-cli session signed in as them) and add their own items under their
own identity. From their own `dd-cli`:

```
dd-cli cart add-items --store-id <id> --menu-id <id> \
  --items-json '[...]' --group-cart-url "https://drd.sh/cart/<code>/" \
  --intent "..."
```

`--group-cart-url` joins-and-adds in one call. Use the `cart_uuid` it
returns for any further adds by that same person
(`--cart-uuid <same_uuid>`, no need to re-pass the URL).

This is the "agentically" path if multiple users in the group cart have access to dd-cli: *"Everyday, look at the
group cart posted in #team-lunch — order me the vegetarian option, and if
there's an avocado add-on, add it"* — the employee's own agent runs this
against the employee's own signed-in `dd-cli`, not the bot's.

### 2. A guest (never signs in to DoorDash at all)

This is what makes the "reply in the Slack thread and the bot adds it for
you" pattern possible. The **bot's own authenticated host account** adds
items into a separate sub-cart tagged with the guest's name — the guest
never touches DoorDash, never authenticates, and doesn't need an account.

```
dd-cli cart add-items --store-id <id> --menu-id <id> \
  --items-json '[{"item_id":"<id>","item_name":"Veggie Burrito","quantity":1}]' \
  --cart-uuid <group_cart_uuid> \
  --guest-json '{"first_name":"Luke","last_name":"Wulf"}' \
  --intent "..."
```

Rules for `--guest-json`, exactly as `dd-cli` enforces them:

- **First add for a new guest**: pass `{"first_name":"...","last_name":"..."}`
  plus `--cart-uuid <existing group cart>` (or `--group-cart-url` if this is
  also the guest's first-ever add to that cart). You cannot combine
  `--guest-json` with `--group-cart` / `--spend-limit-cents` — those only
  apply to *creating* a cart, and a guest add always targets an *existing*
  one.
- **The response** for that call carries a `guest_token` on the acted-for
  sub-cart. **This is the only place that token appears.** `cart show` never
  returns it. Store `cart_uuid` + `guest_token` together (e.g. keyed by
  Slack user id) the moment you get them — if you lose the token, there is
  no way to fetch it again, only to start a new guest with the same name
  (which most people won't have to accept but you should treat as a real
  loss of continuity for line-item tracking).
- **Every later add for the same guest**: pass
  `{"guest_token":"<stored_token>"}` with the same `--cart-uuid` (the
  group cart's canonical uuid — the one from the very first add, not a
  per-call value). Don't resend `first_name`/`last_name` alongside a token.
- **Never surface the token to a human** — not in Slack, not in logs. It's a
  bearer credential for that guest's sub-cart. Keep it server-side in
  whatever your bot uses for state (a small keyed store: Slack user id →
  `{cart_uuid, guest_token}`).
- **Adds are additive, not idempotent.** Before retrying a failed or
  timed-out add, check the response's `item_errors[]` for that attempt — if
  the item isn't listed there, it already succeeded, and a blind retry will
  double it in the cart.

## Setting up the daily cart

### 1. Create the group cart (bot's own identity, once per day)

```
dd-cli cart add-items --store-id <id> --menu-id <id> \
  --items-json '[{"item_id":"<placeholder_or_first_item>","item_name":"...","quantity":1}]' \
  --group-cart --spend-limit-cents <cents_or_omit> \
  --intent "..."
```

- `--group-cart` makes this a shareable cart instead of a personal one.
- `--spend-limit-cents <n>` sets a **per-participant** cap in cents for a
  host-pays-all cart (the bot's payment method covers everyone, each
  participant/guest capped at `n`). **Omit it entirely** for unlimited
  per-participant spending — there is no "unlimited" sentinel value, the
  flag simply isn't passed. The host itself is exempt from its own
  per-participant limit.
- The response's `cart.group_cart_url` is the link to post in the channel.
  It's `null` on a personal (non-group) cart — if you see `null`, the
  `--group-cart` flag didn't take effect; don't post a broken link.

Post `cart.group_cart_url` to the channel from the bot account. With this link, anyone can open it in a browser and
add items themselves without any dd-cli involvement.

### 2. Reading natural-language replies and mapping to guests

For each Slack reply in the thread, resolve the person's display name to a
guest identity:

- **First message from this person today**: create a new guest via
  `--guest-json '{"first_name":"...","last_name":"..."}'` on the group
  cart's `cart_uuid`. Store the returned `guest_token` keyed by their Slack
  user id for the rest of the day's thread.
- **Later message from the same person**: reuse
  `--guest-json '{"guest_token":"<stored>"}'`.
- If you can't confidently parse an item from their message (ambiguous
  dish, missing modifier choice), ask a clarifying reply in-thread before
  calling `cart add-items` — don't guess a menu item id.

### 3. Reminder pass before the deadline

At the stated reminder time, inspect the cart:

```
dd-cli cart show --cart-uuid <group_cart_uuid> --intent "..."
```

`cart show` returns line items but **not** who they belong to by name in a
structured way beyond what you tracked yourself — cross-reference against
your own Slack-user → guest/participant mapping (built in step 2, plus any
real participants who joined via the URL themselves) to figure out who
hasn't added anything, then DM those people the reminder.

### 4. Checkout at the deadline

```
dd-cli order preview --cart-uuid <group_cart_uuid> --intent "..."
dd-cli order submit --cart-uuid <group_cart_uuid> --tip-cents <n> --yes --intent "..."
dd-cli order status --order-uuid <order_uuid> --intent "..."
```

The bot's own account is the one charged (host-pays-all) unless your
company's flow is "everyone pays for themselves," which DoorDash's group
cart does not currently support per-person split billing on — confirm that
assumption with your own ops team before promising split billing to
employees.

Post the confirmation in-channel once `order submit` returns, and ping the
channel again once `order status` reaches its delivered/picked-up terminal
state — poll every ~15 seconds until placed, then back off to every ~15
minutes for delivery tracking.

### 5. Next day

Group carts don't auto-reset. Either `dd-cli cart delete --cart-uuid <id>`
the previous day's cart after checkout (treat it as abandoned — submit
already consumed it) and create a fresh one for the next day, or confirm
your own testing shows carts naturally end at submit; when in doubt, delete
and recreate daily rather than assuming.

## Verifying your bot setup before trusting it with a real order

Before wiring this into a live Slack channel, dry-run the mechanics with
`dd-cli` directly against a cheap, real store, using **two separate
identities**:

1. Sign in as the bot host: `dd-cli login`.
2. Create a group cart with a small `--spend-limit-cents` (e.g. `100` for a
   $1.00 cap) — cheap enough that a rejected over-cap add costs you nothing
   to observe.
3. From a **second** signed-in `dd-cli` session (a different DoorDash
   account — a personal test account, not the bot's), join via
   `--group-cart-url` and try adding an item over the cap. Confirm it comes
   back rejected with a spending-limit `item_errors[]` entry — that's the
   host-pays-all enforcement working as intended for real participants.
4. As the bot host again, add a guest via `--guest-json` with a name (no
   token yet), capture the returned `guest_token`, then add a second item
   for the *same* guest using only `{"guest_token": "..."}`. Confirm
   `cart show` reflects both items under that sub-cart and that `cart show`
   itself never echoes the token back to you.
5. `dd-cli order preview --cart-uuid <id>` to confirm it prices cleanly, then
   `dd-cli cart delete --cart-uuid <id>` to clean up — **do not run
   `order submit`** during this dry run; it's a live store and will charge a
   real card.

If any of steps 2-4 behaves differently than described here, treat this
skill's description as possibly stale against the current DoorDash MCP
contract rather than assuming your bot code is wrong — re-check with
`dd-cli cart add-items --help` for the current flag semantics.

## Common pitfalls

- **Mixing up guest mode and participant mode.** A guest is added by the
  *host's* authenticated `dd-cli` call with `--guest-json`; a participant
  adds themselves with their *own* authenticated `dd-cli` call and
  `--group-cart-url`/`--cart-uuid`. Don't try to use `--guest-json` from a
  non-host identity — only the authenticated group-cart host can add for a
  guest.
- **Losing a guest_token.** It only appears once, on that guest's first-add
  response. Persist it immediately; there's no retrieval endpoint.
- **Retrying a timed-out add without checking `item_errors[]` first** —
  adds are additive, so a blind retry can double an item that actually
  succeeded.
- **Forgetting `--cart-uuid` must be the group cart's canonical uuid** for
  every follow-up guest add — not a value from some other call.
