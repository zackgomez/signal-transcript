---
name: signal-transcript
description: "Pull a scoped markdown transcript from Zack's Signal Desktop DB when he references a Signal conversation, instead of asking him to paste messages"
---

# signal-transcript — read Zack's Signal history directly

When Zack references a Signal conversation ("what did Aaron say", "check my thread with X", "look at the last 10 minutes with Y"), run this instead of asking him to paste messages in.

Run `signal-transcript --help` for full usage — it is the authoritative reference.

Rules for Claude:

- Primary invocation: `signal-transcript "Aaron" --since 10m`. Output is markdown on stdout — read it directly; only use `--out FILE` for unusually large pulls.
- Scope discipline: this reads Zack's private messages. Always scope to the named person and the smallest sensible time window (`--since`, `--last N`). Never dump the whole archive, and don't pull unrelated conversations out of curiosity.
- To read attachment contents (sent files, pasted images), add `--attachments DIR` pointed at the session scratchpad — it decrypts each in-scope attachment to DIR and the transcript line gains the saved path, which you can then Read directly (text and images both). A `(extract failed: …)` annotation means that one attachment couldn't be recovered — usually not yet downloaded by Signal Desktop.
- Ambiguous name match exits 2 and lists candidates on stderr — pick from those or ask Zack, don't retry with a guessed name.
- No match exits 1 and lists recent conversations on stderr; `--list` shows all available conversations by name.
- Read-only against Signal's data and never touches the live DB (it copies it first) — safe to run even while Signal Desktop is open.
- The DB only holds what Signal Desktop has synced. If expected messages are missing, check `pgrep -f /usr/bin/signal-desktop` — if it's not running, launch it on the niri session (`NIRI_SOCKET=$(ls /run/user/1000/niri.*.sock) niri msg action spawn -- signal-desktop`; if no window appears, run `signal-desktop --ozone-platform=wayland` with `WAYLAND_DISPLAY=wayland-1` and no `DISPLAY`), give it a minute to sync, and re-pull.
- If stderr shows a `Signal schema changed` warning or error, a Signal Desktop update moved something — tell Zack rather than silently handing back a possibly-degraded transcript.
