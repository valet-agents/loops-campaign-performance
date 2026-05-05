This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **loops**: The Loops CLI, exposed as the bare command `loops` on PATH (the connector is named to match the command, not the npm package). The agent uses it to list campaigns and sequences, pull aggregate metrics (sends, opens, clicks, replies, unsubscribes), and — only when a Slack user explicitly asks and confirms — trigger transactional sends or edit contact list membership. Add it from the catalog at the org level so other Loops-powered agents can share it.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, and posts the daily campaign digest to whichever channels the bot has been invited to. Slack writes use the auto-injected outbound Slack connector.
- **cron** (cron): Fires the daily campaign performance digest at 8am Pacific, Monday through Friday (`0 8 * * 1-5`, `America/Los_Angeles`). Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow.

### Secrets

- **LOOPS_API_KEY**: A Loops API key. Create one at [app.loops.so/settings/api](https://app.loops.so/settings/api). A read-only key is enough for the daily digest and ad-hoc Q&A. Use a read-write key only if you want the agent to send transactional emails or edit contact lists on request.

### External Setup

1. Generate a Loops API key at app.loops.so/settings/api and paste it into the connector's `LOOPS_API_KEY` slot in the Valet dashboard.
2. After deploy, invite the agent's Slack bot to whichever channel(s) you want the daily digest in. The agent posts to every channel it's a member of — invite it to one growth channel, or several. If the bot has not been invited anywhere, the digest is sent as a DM to the workspace install user with a one-line nudge to invite it somewhere.
3. Invite the bot to any additional channels where teammates should be able to @mention it for ad-hoc questions (e.g. *"how's the Q2 nurture sequence performing?"*).
4. The first cron fire is the next 8am Pacific weekday after deploy. To smoke-test sooner, @mention the bot in Slack with a question like *"which campaign has the highest reply rate this month?"* — that exercises the Slack + Loops path without waiting for the cron.

## Customizing

- **Change the schedule**: edit the `cron` and `timezone` on the `cron` channel in `valet.yaml`, then redeploy. The default `0 8 * * 1-5` `America/Los_Angeles` is "8am Pacific, Monday through Friday."
- **Change the "quietly dying" drop threshold**: the SOUL flags a sequence when its 7-day reply rate dropped more than 30% versus the prior week (and the prior week was at least 1%). Edit the threshold in `SOUL.md` → "Phase 3: Detect quietly dying sequences" if you want a different sensitivity.
- **Control where the digest posts**: invite or remove the bot from channels in Slack — that's the only signal the agent uses. There is no channel name in the configuration.
- **Read-only vs read-write**: use a read-only Loops API key if you want to forbid the agent from ever sending transactional email or editing contacts, even on confirmation. The agent will surface the CLI's permission error verbatim if a write is attempted.
