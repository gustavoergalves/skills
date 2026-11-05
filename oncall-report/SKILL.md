---
name: "oncall-report"
description: "Write my short backend on-call report from the Slack threads and notes I pass (bugs channel, dd-alerts, anything else I watched). Use when I share on-call threads with outcomes, or ask for an on-call / guard-duty report."
argument-hint: "Slack thread links with outcomes, plus free-text notes"
user-invocable: true
disable-model-invocation: false
---

# On-call report

Turn what I pass about my backend on-call day into a short report: one line per thing that happened, what it was and
how it ended. Same spirit as the standup: no details, no explanations.

## Input

I pass it in the message or `$ARGUMENTS`, usually like this:

```
bugs channel: https://zendropapp.slack.com/archives/CHXMWAW6R/p1793767946881879
outcome: vadim khorava (the legend) took care of it. no bug on our side.

dd-alerts: https://zendropapp.slack.com/archives/C08FDEJ3E78/p1793784163789819
outcome: created the bug ticket BE-848

redis consumption alerts: in the evening, around 9:30pm brt, redis started sending high connection count alerts on dd-alerts.
```

- A link plus an outcome, or a free-text note with no link.
- Known channels: `CHXMWAW6R` is #bugs, `C08FDEJ3E78` is #dd-alerts. Otherwise use the name I wrote.

## Output

Reply with only the report, in English unless I ask otherwise, in this shape:

```
**Backend on-call, Wed Nov 4**
- [#bugs](https://zendropapp.slack.com/archives/CHXMWAW6R/p1793767946881879): order not syncing from Shopify → Vadim Khorava (the legend) took care of it, no bug on our side
- [#dd-alerts](https://zendropapp.slack.com/archives/C08FDEJ3E78/p1793784163789819): Shopify hold-fulfillment webhook 500 on `markets_pro` → created BE-848
- #dd-alerts: Redis high connection count alerts from ~9:30pm BRT
```

- One line per item: `channel: what → outcome`. The channel links to the thread when I passed a link.
- **what**: 3–8 words saying what the report or alert was. No client names, store IDs or order numbers.
- **outcome**: my wording, only tightened (fix capitalization and names, drop filler). Keep credits, jokes and ticket
  keys.
- No outcome given: write only the observation, with no arrow. Never invent an outcome.
- Items in the order I passed them.
- Nothing passed at all: a single line, `- quiet day, nothing on our side`.
- No text before or after the report. Do not post it to Slack.

## Steps

1. **Date.** Use the date I give. Otherwise use the date of the linked messages in `America/Sao_Paulo` time
   (`TZ=America/Sao_Paulo date -r <seconds>`), otherwise today.
2. **Read each linked thread**, all in one parallel batch, with
   `mcp__plugin_slack-by-salesforce_slack__slack_read_thread` (deferred: load it with ToolSearch first),
   `response_format: "concise"`. From a link `.../archives/<CHANNEL>/p<DIGITS>`, `channel_id` is `<CHANNEL>` and
   `message_ts` is `<DIGITS>` with a dot before the last 6 digits (`p1793767946881879` → `1793767946.881879`).
3. **Write the "what"** from the parent message. Datadog alerts often arrive with an empty body; use the replies then.
4. **Notes without a link** become one line from my note as written: keep the times and timezone I gave. Do not search
   Slack or Datadog for them.
