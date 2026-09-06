# Beeper (messaging aggregator ingestion)

- **category**: messaging
- **provider**: Beeper (via `mcp__beeper__*` MCP tools / Beeper Desktop)
- **reusable**: partial — the ingestion *pattern* (durable-id resolution, dedup key, image decrypt-and-cache handling) transfers cleanly to any repo; the actual pull depends on the user's own Beeper Desktop/MCP setup, so it isn't an installable package, just a recipe.
- **docs**: https://developers.beeper.com/ (Beeper Desktop API / MCP tools)

## Overview
Beeper Desktop bridges many chat networks (WhatsApp, Facebook Messenger, iMessage, etc.) into one local client and exposes them to an agent via MCP tools (`search_chats`, `get_chat`, `list_messages`, `focus_app`, ...). This lets an agent pull messages/attachments from any bridged network without touching that network's own API — useful for one-off or scheduled ingestion of free-text or link-share content from chats/groups into an app's own data store.

## Playbook

### Prerequisites
- Beeper Desktop installed and logged into the networks you want to read (WhatsApp, Messenger, etc.), with the MCP server exposed to the agent (`mcp__beeper__*` tools).
- The target chat(s) already bridged/joined in Beeper Desktop — the agent can search but can't join a network on your behalf.
- A destination data store with its own dedup key (e.g. a `sourceUrl`-unique constraint) — Beeper gives you no delivery/read receipts to lean on for idempotency.

### Setup steps
1. **Register each source with a durable identifier**, not Beeper's numeric chat id. Beeper's numeric `chatID` (what `search_chats`/`get_chat`/`list_messages` accept) is a local-index alias that renumbers over time — confirmed empirically: hardcoded numeric ids went stale (404) within ~2.5 months. Store the *type* of source (`single` = personal contact / DM, `group` = a real group chat) alongside a human-searchable name, and resolve the live chat defensively at the start of every run:
   - Call `get_chat({ chatID: storedId })`.
   - On 404, re-resolve via `search_chats({ query: <distinctive name token>, type: "group" | "single", scope: "participants" for singles })`, confirm the match's name/network genuinely matches (don't take the first hit blindly), and persist the freshly resolved id back to your own config so the next run hits cache directly.
2. **Pull recent messages** with `list_messages({ chatID })`, paginating `direction: "before"` from the newest page until messages fall outside your lookback window. An empty result shaped like `{ items: [], hasMore: true, oldestCursor: null, newestCursor: null }` means the chat has zero synced history (e.g. a just-joined group — most networks don't backfill pre-join history) — treat as "no messages yet", not an error.
3. **Clean message text** before parsing: raw `text` fields can contain literal `<br>` (and occasional `<strong>`) tags from the bridge — convert to newlines and strip other tags first.
4. **Branch extraction by chat type**: a `single`/DM source is typically link-shares (scan for URLs, then resolve/fetch the linked content); a `group` source is typically free-text posts requiring your own content gates (skip deleted/retracted messages, skip off-topic posts, etc.) and field-by-field text parsing.
5. **Build a dedup key from durable ids, never the numeric chat id** — synthesize something like `synthetic://<message.chatID>/<message.id>` using the message's own `chatID` field (a durable per-network id embedded in every message, distinct from the numeric lookup id) plus its own message `id`. Feed this into your store's existing dedup logic (e.g. unique-by-URL) so re-running the ingestion is always safe.
6. **Handle attachments' decrypt-then-cache lifecycle**: an attachment's `srcURL` starts as an opaque encrypted-media reference; if it's already a local `file://` path, use it directly. If it's not yet cached, call `focus_app({ chatID, messageID })` to bring that message into view in Beeper Desktop, wait a few seconds, then re-fetch the message — Beeper Desktop decrypts and caches the attachment locally once focused, turning the reference into a plain `file://` path. Don't attempt to hand-roll the underlying E2E decryption yourself; if it's still not cached after one retry, skip just that attachment and note it, rather than blocking the whole batch.
7. **Upload resolved local files** to your own storage (URL-decode the `file://` path, read raw bytes, upload) instead of re-fetching over HTTP — the file is already local.

### Core pattern
```python
#!/usr/bin/env python3
"""Generic pattern: upload locally-cached Beeper attachment files to your
own object storage, once you have (target_id, [file:// paths]) pairs."""
import urllib.parse

def resolve_local_path(src_url: str) -> str | None:
    """A message attachment's srcURL, once cached, looks like
    'file:///Users/.../BeeperTexts/media/<network>/<hash>.jpg'.
    Returns a filesystem path, or None if still an opaque (uncached) reference."""
    if not src_url.startswith("file://"):
        return None  # not yet cached — re-focus the message and retry once
    return urllib.parse.unquote(src_url[len("file://"):])

# Fill in from your own ingestion loop: target record id -> list of resolved paths
attachments_by_target: dict[str, list[str]] = {
    # "some-record-id": ["/Users/.../BeeperTexts/media/local.beeper.com/xyz.jpg"],
}

for target_id, paths in attachments_by_target.items():
    for path in paths:
        with open(path, "rb") as f:
            data = f.read()
        # upload_to_your_storage(target_id, data)   # e.g. Firebase Storage, S3, etc.

# Dedup key builder — never use the numeric chatID for this:
def make_dedup_key(message_chat_id: str, message_id: str) -> str:
    return f"synthetic://{message_chat_id}/{message_id}"
```

### Env vars
None — Beeper is accessed entirely through its own MCP server / Desktop app, not through this app's environment configuration.

### Gotchas
- Never persist Beeper's numeric `chatID` as a permanent identifier — it silently renumbers over time (observed drift within ~2.5 months). Always resolve defensively at the top of a run and re-persist on drift.
- The dedup key must come from the message's own embedded `chatID` field (a durable per-network id) plus its `id`, not the numeric lookup id used to fetch the chat.
- `{ items: [], hasMore: true, oldestCursor: null, newestCursor: null }` from `list_messages` is a legitimate "no history synced yet" state, not an error — don't retry it as a failure.
- Attachments go through a decrypt-and-cache step controlled by Beeper Desktop itself (`focus_app` on the message triggers it); there is no way to fetch/decrypt Matrix-style encrypted media directly from the MCP layer.
- Because there are no delivery/read receipts to lean on, correctness depends entirely on your own destination-side dedup (unique constraint on a stable key) — always design that before writing the ingestion loop, not after.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace, which pulls new records from a mix of bridged single-contact and group chats, saving parsed results through its own tooling and fetching/decrypting locally-cached message images for upload to its own object storage.
