The Loops CLI is named `loops` and is on PATH. It uses the `LOOPS_API_KEY` env var (already injected at runtime — never echo it). Use it for read-only queries about campaigns, transactional templates, mailing lists, contacts, and events. Default to read; only send a transactional email or modify a contact list when the SOUL workflow explicitly tells you to (and only after Slack confirmation).

─── Running commands non-interactively ──────────────────────────────────────────────────────────────

Always invoke the CLI with this shape:

  PAGER=cat loops <root-flags> <resource> <subcommand> <subcommand-flags>

Three rules that bite if you get them wrong:

1. Disable the pager. When stdout is a TTY the CLI may pipe long output through `$PAGER` (default `less`) and hang waiting for keypresses. Prefix every invocation with `PAGER=cat` (or pipe to `| cat`).

2. Root flags go BEFORE the resource, subcommand flags go AFTER.
   `--format`, `--api-key`, `--debug`, `--yes` / `-y` are root flags on `loops` itself — placing them after the subcommand makes them unknown flags. The `--api-key` is auto-injected from `LOOPS_API_KEY`; you should not pass it yourself.

3. List before fetch. Always `list` campaigns / templates / lists first to capture stable IDs, then pass an explicit `--id <uuid>` (or `--campaign-id`) to the metrics subcommand. Names can collide, IDs cannot.

Canonical example — list campaigns as JSON:

  PAGER=cat loops --format json campaigns list

For one campaign's aggregate metrics yesterday:

  PAGER=cat loops --format json campaigns metrics --id <campaign-id> --since yesterday --until today

For per-recipient events on the same campaign (line-delimited JSON, friendlier for piping):

  PAGER=cat loops --format jsonl campaigns events --id <campaign-id> --since yesterday

// TODO: confirm exact subcommand names against `loops --help` at runtime — the CLI exposes a small set of resources (campaigns, transactional, contacts, lists, events) but the verb naming (`metrics` vs `stats` vs `report`) is worth verifying with `loops campaigns --help` on first use.

─── What you can ask Loops for ──────────────────────────────────────────────────────────────────────

Aggregate metrics per campaign / sequence:

- **sends** — total recipients delivered to in the window
- **opens** — unique opens (and open rate = opens / sends)
- **clicks** — unique clicks (and click rate = clicks / sends)
- **replies** — replies attributed to the campaign (only useful if reply tracking is on)
- **unsubscribes** — opt-outs from this campaign in the window
- **bounces** — hard + soft, broken out if the CLI exposes it

Per-recipient events (one row per send):

- recipient email + contactId
- timestamps for `delivered`, `opened`, `clicked`, `replied`, `unsubscribed`, `bounced`
- the link URL for click events

By-campaign breakdowns are the default — `campaigns list` returns one row per campaign or sequence step. To roll up a multi-step sequence (e.g. "Q2 Nurture" with 5 emails), list campaigns, filter by name prefix or `sequence_id`, then sum the per-step metrics yourself.

─── Scoping a query ─────────────────────────────────────────────────────────────────────────────────

Always scope by ID, not by name, once you have it:

  # 1. Find the campaign
  PAGER=cat loops --format json campaigns list --transform 'campaigns.#(name%"*Q2 Nurture*")'

  # 2. Pull metrics for that ID
  PAGER=cat loops --format json campaigns metrics --id <id-from-step-1> --since 2026-04-01 --until 2026-05-01

Time windows: prefer absolute ISO dates (`--since 2026-05-04 --until 2026-05-05`) over relative words. Yesterday-only is `--since <yesterday-iso> --until <today-iso>`.

Active vs all: most `list` subcommands accept `--status active` (or `--state running` — confirm at runtime). Use it on the cron path to skip drafts and archived campaigns.

─── Computing rates ─────────────────────────────────────────────────────────────────────────────────

The CLI usually returns counts, not percentages. Compute and quote rates yourself, rounded to one decimal place:

  open_rate   = opens   / sends   * 100
  click_rate  = clicks  / sends   * 100
  reply_rate  = replies / sends   * 100
  unsub_rate  = unsubs  / sends   * 100

Skip the rate (show `—`) when sends < 20 in the window — the denominator is too small to be meaningful.

For the "quietly dying" comparison, fetch two windows in parallel: the trailing 7 days, and the 7 days before that. Flag a sequence when this-week reply rate dropped > 30% relative to last-week reply rate AND last-week reply rate was at least 1% (avoid flagging campaigns that were already at zero).

─── Writes (only on confirmation) ───────────────────────────────────────────────────────────────────

These mutate state. Never run them on the cron path. On the Slack path, only after the user confirms in-thread.

  # Send a one-off transactional email
  PAGER=cat loops transactional send --template-id <id> --email <to> --data '{"name":"…"}'

  # Add or remove a contact from a list
  PAGER=cat loops contacts update --email <to> --add-list <list-id>
  PAGER=cat loops contacts update --email <to> --remove-list <list-id>

Every write should include `--yes` to skip the CLI's own confirm prompt — the user's Slack 👍 is the only confirmation that matters.

─── Output etiquette ────────────────────────────────────────────────────────────────────────────────

- Always pass `--format json` (or `jsonl`) so output is parseable. Default text output is for humans and will mislead you.
- Use `--transform '<gjson>'` to narrow large list payloads on the CLI rather than reading the whole blob. Saves tokens.
- If a command errors (non-zero exit), surface the error message in your Slack reply verbatim — do not invent a friendlier version. Loops's error text is short and clear.
- Never paste `LOOPS_API_KEY` into a message or include it in any output transform.
