# DoorDash MCP — Agent Skills

Drop-in skills that teach an AI coding/ops agent (Claude Code, Codex, or any
harness that reads plain Markdown "skill" files) how to use the **DoorDash
Corporate Ordering MCP** — via the open-source `dd-cli` wrapper — for the top
use cases companies ask for once the DoorDash MCP connector is live for their
employees.

These skills assume:

1. Your company has been approved for DoorDash Corporate Ordering MCP access,
   and the connector (or `dd-cli`) is set up.
2. `dd-cli` is installed and on `PATH`. See
   https://github.com/doordash/doordash-cli for install instructions.
3. Each employee (or the bot's service identity) runs `dd-cli login` once —
   it opens a browser, signs in with their own DoorDash account, and saves
   the credentials to the OS keychain. Every order placed carries that
   person's own DoorDash identity; `dd-cli` never asks for or stores a
   password.

For a headless environment (a Slack bot process, a CI job, a server with no
browser) run `dd-cli export-token` once **on a machine with a browser**, then
set the printed value as the `DD_CLI_ACCESS_TOKEN` environment variable in
the headless environment. `dd-cli` prefers that env var over the keychain
whenever it's set. Treat the token like a password — secrets manager only,
never in code or chat logs.

## What's in here

| Skill | Use case from the 1-pager | Best for |
|---|---|---|
| [`dd-lunch-autopilot`](skills/dd-lunch-autopilot/SKILL.md) | "Put lunch/dinner on autopilot" | One employee automating their own recurring order |
| [`dd-quick-order`](skills/dd-quick-order/SKILL.md) | "Order within the surfaces you already do work" | Ad-hoc one-off orders dropped into any agent session |
| [`dd-team-recurring-order`](skills/dd-team-recurring-order/SKILL.md) | "Set up a recurring routine on behalf of your team" | One person sets up a standing team lunch / all-hands treats order |
| [`dd-group-cart-slackbot`](skills/dd-group-cart-slackbot/SKILL.md) | "Set up a company ordering agent / Slack bot" (group orders) | A shared bot that runs a daily/weekly **group cart**, lets employees add their own items (manually, by DM, or agentically), and checks out on a deadline |
| [`dd-office-supplies-concierge`](skills/dd-office-supplies-concierge/SKILL.md) | "Office supplies and snacks" Slack bot | A bot that accumulates ad-hoc requests into one cart all week and checks out at a fixed time |

`dd-group-cart-slackbot` is the deep one — it documents the **guest sub-cart
mechanism** (`--guest-json` / `guest_token`) that lets a single authenticated
host account hold and price separate line-item groups for people who never
sign in to DoorDash themselves. Read it even if you're building something
custom, since every other group-ordering pattern is a variation on it.

## Loading these into your agent

**Claude Code / Claude Agent SDK:** copy the `skills/` directory (or the one
subfolder you want) into your project's `.claude/skills/`, or into
`~/.claude/skills/` for a user-level skill available in every project. Claude
Code will list it in `/skill` and load it automatically when the request
matches its `description`.

**Codex / other agents:** these are plain Markdown with a YAML frontmatter
header (`name`, `description`) and no Claude-specific tool syntax — every
command shown is a literal shell command (`dd-cli ...`). Point your agent's
system prompt or project instructions at the relevant `SKILL.md` file(s), or
concatenate the ones you need into your `AGENTS.md` / custom instructions.
Nothing in here depends on a specific agent framework.

## A note on `--intent`

Every `dd-cli` command that talks to DoorDash requires an `--intent` flag —
a one-sentence plain-language statement of the goal behind the call (who it's
for and why), not a restatement of the command. All example commands below
include one; keep that habit when you adapt them.
