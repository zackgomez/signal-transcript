# signal-transcript — backlog

Ideas not yet built. The tool itself is `.local/bin/signal-transcript`; the Claude-facing
usage contract is `.claude/skills/signal-transcript/SKILL.md`.

## Delayed-send timestamp annotation

Transcripts are ordered by `received_at` (Signal's arrival counter), but each line is
labelled with `sent_at` (the sender's clock). When a message was queued before delivery —
an outage, travel, a phone offline — its label reads earlier than the lines above it:

```
**13:42 Aaron Sharpe:** I can certainly recruit more help tomorrow but we lose a day
**13:12 Zack:**
  [attachment: image/jpeg — signal-2026-08-02-131229.jpeg]
**13:44 Zack:** It took you all week to make ur talk
```

The order is correct and matches Signal Desktop; only the label looks wrong. Proposal is to
render delayed rows as `**13:12→13:43 Zack:**` when `received_at_ms - sent_at` exceeds ~60s,
so the backwards jump explains itself instead of reading as corruption. Deferred — the
annotation is display noise on every normal day, and delayed sends are rare.

## Reactions and edits do not tail

A reaction lands in the `reactions` table without touching the target message's
`received_at`; an edit rewrites a row in place. A cursor-based `--after` pull therefore
cannot surface either. Accepted as-is: 34 reactions against 27.5k messages.

## Day headers across tail pulls

`render_messages` emits a `### date` header only when the day changes *within* one pull, so
a `--after` tail that crosses midnight never prints one. Would need the cursor to carry the
last rendered day.

## Wake policy for a following Claude

Tailing with `--after` in a `Monitor` loop wakes the reader on every message, which for the
Aaron thread is ~87 wakes/day against a median inter-message gap of 14s — the conversation
arrives in bursts, so nearly every wake carries one line.

Batching on a quiet period, with a message cap and a max-wait so a fast back-and-forth
cannot stall the batch, and triggering only on incoming (while still delivering Zack's own
messages as context, since a reply is unreadable without the question):

```
policy                                    /day  msgs/wake  med lat  p95 lat   worst
per-message                                 87        1.0       0s       0s      0m
quiet 60s + cap 10 + max 5m, incoming       18        4.8      20s     127s      3m
quiet 120s + cap 12 + max 5m, incoming      14        6.1      45s     256s      6m
quiet 300s + cap 20 + max 15m, incoming     10        8.5      98s     609s     15m
```

Measured over 2,611 messages across 30 days. quiet 120s / cap 12 / max 5m is the pick.

This belongs in the watcher loop, not in the tool: `--after` already returns the batch and
the rendered line names the speaker, so the policy is ~12 lines of shell and stays tunable
per conversation. A `--follow` mode with a fixed debounce baked in would be strictly worse.
Revisit only if one set of numbers turns out to fit every conversation.
