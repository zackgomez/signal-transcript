# signal-transcript

Extract scoped, readable transcripts from Signal Desktop's local message database.

Signal Desktop keeps its history in an SQLCipher-encrypted SQLite file. This reads it and
renders markdown — a conversation, a time window, attachments decrypted to disk — so you can
grep your own messages, feed a thread to an LLM, or follow a live conversation without
copy-pasting screenshots.

```
$ signal-transcript "Aaron" --since 30m
# Signal — Aaron Sharpe
_last 30 minutes · 2026-08-21 22:11–22:15 EDT · 5 messages_

**22:11 Zack:** is that all you think needs to be changed?
**22:13 Aaron Sharpe:** just send another rev but should begood
**22:15 Aaron Sharpe:** lgtm

_cursor: 1772999891276_
```

## Install

Single self-contained script with a [PEP 723](https://peps.python.org/pep-0723/) header, so
[uv](https://docs.astral.sh/uv/) fetches its dependencies on first run. Nothing to build.

```sh
ln -s "$PWD/signal-transcript" ~/.local/bin/
signal-transcript --list
```

macOS and Linux. `signal-transcript --help` is the authoritative usage reference.

## Following a conversation

Every run ends with a `cursor:` line — the highest arrival counter it covered. Pass it back
as `--after` to get only what has landed since:

```sh
$ signal-transcript "Aaron" --after 1772999891276
_no new messages · cursor: 1772999891276_
```

That makes polling cheap enough to run in a loop, and is why the tool is useful to an agent
following a live thread: an idle check costs one line instead of re-emitting the window.

## Safety

Read-only against Signal's data by construction. The live database is never opened — it is
copied to a temp dir first, so the tool is safe to run while Signal Desktop is open. The only
writes are the transcript (`--out`) and decrypted attachments (`--attachments DIR`).

## How it works

- **DB key.** Linux stores it in plaintext in `config.json`. macOS wraps it with Electron
  `safeStorage`, which is Chromium `os_crypt`: a Keychain password stretched by PBKDF2-SHA1
  (salt `saltysalt`, 1003 rounds) into an AES-128 key, decrypting `v10`-prefixed AES-128-CBC.
- **Ordering.** Rows are ordered by `messages.received_at`, Signal's monotonic insert counter
  and the leading key of its own conversation indexes — not by `sent_at`, which is the
  sender's clock and reorders under delayed delivery or clock skew. Displayed times remain
  `sent_at`, matching what Signal shows.
- **Attachments.** Version-2 files on disk are `IV(16) || AES-256-CBC || HMAC-SHA256(32)`,
  keyed by the per-attachment 64-byte `localKey` (AES key ‖ MAC key), verified against the
  recorded plaintext hash.

None of this is a published interface. A Signal Desktop update can move any of it; the tool
warns when a table it needs has gone missing rather than returning a quietly degraded
transcript.

## License

MIT
