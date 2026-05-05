# Daily Campaign Performance Digest

The cron channel fires once on its schedule (8am Pacific weekdays).
There is no payload to parse — your job is to run the digest
workflow and post the result to Slack.

## Steps

1. Follow the **Daily Performance Workflow** in SOUL.md (Phases
   1–5): list active campaigns from Loops, pull yesterday's
   aggregate metrics for each, detect "quietly dying" sequences by
   comparing 7-day reply rate week over week, and compose the
   digest message.
2. Resolve target channels per the SOUL **Where to post** section:
   list every channel the bot is a member of and post once to each.
   If the bot is in zero channels, DM the workspace install user
   instead with the digest and a one-line invite hint.
3. Post exactly once per resolved destination. Do not retry on
   failure — log the error in your session and continue with the
   remaining destinations. The next cron fire is the recovery.
4. Do not send any follow-ups, reactions, or thread replies after
   the initial post. Your turn ends after the posts complete.

## Skip conditions

- **No active campaigns found in Loops**: stop silently. No digest,
  no DM, no "nothing to report" message. The next fire will retry.
- **No campaigns sent yesterday** (campaigns exist but zero sends
  in the window): post a single line `No campaigns sent yesterday.
  Nothing to report.` to the resolved destinations and stop.
- **`LOOPS_API_KEY` is missing or unauthorized** (the CLI returns a
  401 / "missing API key" error): DM the workspace install user
  with a one-liner: *"My Loops API key is missing or invalid. Add
  one at app.loops.so/settings/api and paste it into the connector
  in the Valet dashboard."* Do not post the digest anywhere else.
