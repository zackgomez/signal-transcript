# signal-transcript — backlog

Ideas not yet built. The tool is `signal-transcript`; the Claude-facing usage contract is
`skills/signal-transcript/SKILL.md`.

## Delayed-send timestamp annotation

Transcripts are ordered by `received_at` (Signal's arrival counter), but each line is
labelled with `sent_at` (the sender's clock). When a message was queued before delivery —
an outage, travel, a phone offline — its label reads earlier than the lines above it:

```
**13:42 Sam Okafor:** I can push the deadline a day if that helps
**13:12 Dana Reyes:**
  [attachment: image/jpeg — signal-2026-08-02-131229.jpeg]
**13:44 Dana Reyes:** that works, sending the draft now
```

The order is correct and matches Signal Desktop; only the label looks wrong. Proposal is to
render delayed rows as `**13:12→13:43 Dana Reyes:**` when `received_at_ms - sent_at` exceeds
~60s, so the backwards jump explains itself instead of reading as corruption. Deferred — the
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

Tailing with `--after` in a `Monitor` loop wakes the reader on every message, which for
one active thread is ~87 wakes/day against a median inter-message gap of 14s — the conversation
arrives in bursts, so nearly every wake carries one line.

Batching on a quiet period, with a message cap and a max-wait so a fast back-and-forth
cannot stall the batch, and triggering only on incoming (while still delivering the user's
own messages as context, since a reply is unreadable without the question):

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

## Test suite

There is none, and schema drift is the tool's main failure mode — it reads an undocumented
SQLCipher schema belonging to an app that auto-updates.

Signal's DB has 57 tables; the tool touches 5 (`messages`, `conversations`, `reactions`,
`message_attachments`, `callsHistory`), whose combined DDL is ~7.8 KB and contains no data.
So the fixture is a generated SQLCipher database built from that schema and populated with
fabricated messages — publishable, runnable in CI, and a real test of the contract.

Worth covering:

- **Schema contract** — every column the tool selects exists and holds the type it assumes.
- **Cursor invariants** — `received_at` is non-null and non-decreasing in insert order; a
  chain of `--after` pulls over a fixture is contiguous, with no gaps and no repeats.
- **Ordering** — a message whose `sent_at` precedes its predecessor's still renders in
  arrival order, and a `--since` window admits it when only `received_at_ms` is in range.
- **Attachment crypto** — round-trip a synthesized `v2` blob: encrypt a known payload under
  a fabricated `localKey`, then assert the decrypt, the HMAC rejection on a flipped bit, and
  the plaintext-hash mismatch path.
- **Key extraction** — the Linux plaintext path, and the macOS `os_crypt` unwrap against a
  fabricated Keychain secret.

No test may touch a real Signal database. A separate opt-in check can verify the local
install still matches, asserting shapes only and never message content.

## --doctor

Preflight and drift check, as a flag on the tool rather than an install script: the script
already knows how to locate the Signal directory per platform, extract the key two different
ways, and probe the schema, and a shell installer would reimplement all of it and drift.

Should report: `uv` present; Signal Desktop installed and its directory found; the DB key
extractable (naming which path, and warning about the macOS double dialog); the DB readable;
`~/.local/bin` on `PATH`; every table and column the tool selects present.

The useful part is version drift. `PRAGMA user_version` is Signal's migration counter — 1770
as of 2026-08-26. Recording the highest tested value lets `--doctor` say "Signal has migrated
to 1784; tested through 1770" *before* a query fails mid-transcript, which is the failure
anyone running an old copy of this script will hit first.
