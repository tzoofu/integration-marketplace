# Telegram Bot (python-telegram-bot, long polling)

- **category**: messaging
- **provider**: Telegram (Bot API via `python-telegram-bot` v20, async)
- **reusable**: yes — the bootstrap, namespaced callback routing, toggle-keyboard, consent/privacy commands and per-user localization helper are domain-free and would extract cleanly into a shared starter package; only the command set and data model are app-specific.
- **docs**: https://core.telegram.org/bots/api · https://docs.python-telegram-bot.org/en/v20.7/

A chat-first user interface built as a Telegram bot: users interact through slash commands, inline-keyboard buttons, free text and file uploads, and the bot replies in their chosen language. Reach for it when you want a zero-install UI for a small user base, with no web frontend and no public HTTP endpoint (long polling).

## Playbook

### Prerequisites
- A bot token from **@BotFather** (`/newbot`). Set the command menu there too (`/setcommands`) or call `bot.set_my_commands` at startup.
- Python 3.11+ (3.12-slim image observed), `python-telegram-bot==20.7`, `python-dotenv`.
- Optional: a persistence store for per-user preferences (Firestore via `firebase-admin` observed; see [database/firestore.md](../database/firestore.md)), and `schedule` for in-process retention jobs.
- Docker + Compose for deployment on any small VM. Long polling needs **no inbound ports**.

### Setup steps
1. Create the bot with BotFather, then put the token in `.env` as `TELEGRAM_BOT_TOKEN`. Never `sed` it into source, and never print it in full (log a short prefix at most).
2. Build the app with `Application.builder().token(token).build()`, then register handlers in this order: commands → document `MessageHandler` → text `MessageHandler(filters.TEXT & ~filters.COMMAND)` → `CallbackQueryHandler`s.
3. **Give every `CallbackQueryHandler` a `pattern=`** with a namespaced prefix (`item:toggle:`, `privacy:`). A pattern-less callback handler swallows every callback registered after it in the same group.
4. Register a global error handler (`app.add_error_handler`) and call `logging.basicConfig`. Without them, handler exceptions and service logs are silently dropped.
5. Start with `app.run_polling(allowed_updates=Update.ALL_TYPES, drop_pending_updates=True)`. PTB handles SIGINT/SIGTERM itself. Stop background jobs in `post_shutdown` (or a `try/finally` around `run_polling`).
6. Multi-step input: either a `ConversationHandler`, or a simple flag in `context.user_data` (e.g. `awaiting_input=True`) that the generic text handler checks and clears.
7. Privacy (if you store anything per user): an opt-in consent screen with Accept/Decline/Policy buttons; `/privacy` (show stored data), `/dataexport` (JSON), `/deletedata` (confirm-then-delete), and withdraw-consent. Without consent, keep state in memory only. Key stored records by an HMAC of the Telegram user id, not the raw id.
8. Containerize: a non-root user, exec-form `CMD ["python3", "main.py"]` so SIGTERM reaches Python as PID 1, `restart: unless-stopped`, CPU/memory limits, `no-new-privileges`. Run **exactly one** replica per token.

### Core pattern

**Bootstrap + handler registration**
```python
import logging, os
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import (Application, CommandHandler, MessageHandler,
                          CallbackQueryHandler, ContextTypes, filters)

load_dotenv()
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"),
                    format="%(asctime)s %(name)s %(levelname)s %(message)s")
log = logging.getLogger("bot")

async def on_error(update: object, context: ContextTypes.DEFAULT_TYPE):
    log.exception("Unhandled error", exc_info=context.error)
    if isinstance(update, Update) and update.effective_message:
        await update.effective_message.reply_text("Something went wrong. Please try again.")

async def post_shutdown(app: Application):
    stop_background_jobs()

def main():
    token = os.getenv("TELEGRAM_BOT_TOKEN")
    if not token:
        raise SystemExit("TELEGRAM_BOT_TOKEN is not set")
    app = Application.builder().token(token).post_shutdown(post_shutdown).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("items", list_items_cmd))
    app.add_handler(CommandHandler("privacy", privacy_cmd))
    app.add_handler(CommandHandler("deletedata", delete_data_cmd))

    app.add_handler(MessageHandler(filters.Document.ALL, on_document))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, on_text))

    app.add_handler(CallbackQueryHandler(on_item_toggle, pattern=r"^item:toggle:"))
    app.add_handler(CallbackQueryHandler(on_privacy_cb, pattern=r"^privacy:"))
    app.add_error_handler(on_error)

    app.run_polling(allowed_updates=Update.ALL_TYPES, drop_pending_updates=True)

if __name__ == "__main__":
    main()
```

**Inline-keyboard multi-select (toggle) with re-render**
```python
from telegram import InlineKeyboardButton, InlineKeyboardMarkup
from telegram.error import BadRequest

def build_items_keyboard(items, selected_ids, group_id):
    rows = [[InlineKeyboardButton(
        f"{'✅ ' if it['id'] in selected_ids else ''}{it['name']}",
        callback_data=f"item:toggle:{group_id}|{it['id']}")] for it in items]
    return InlineKeyboardMarkup(rows)

async def list_items_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    group_id = get_selected_group(uid)
    await update.effective_message.reply_text(
        t(uid, "pick_items"),
        reply_markup=build_items_keyboard(load_items(group_id), get_selected_ids(uid, group_id), group_id))

async def on_item_toggle(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    uid = update.effective_user.id
    try:
        group_id, item_id = q.data.removeprefix("item:toggle:").split("|", 1)
    except ValueError:
        await q.answer(t(uid, "bad_request"), show_alert=True)
        return
    added = toggle_selection(uid, group_id, item_id)
    await q.answer(t(uid, "item_added" if added else "item_removed"))
    try:
        await q.edit_message_reply_markup(
            build_items_keyboard(load_items(group_id), get_selected_ids(uid, group_id), group_id))
    except BadRequest as e:
        if "not modified" not in str(e).lower():
            raise
```

**Confirm-then-delete + pseudonymous storage key**
```python
import hashlib, hmac

def pseudonymize(user_id: int) -> str:
    salt = os.environ["USER_ID_SALT"]
    return hmac.new(salt.encode(), str(user_id).encode(), hashlib.sha256).hexdigest()

async def delete_data_cmd(update, context):
    uid = update.effective_user.id
    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton(t(uid, "yes_delete"), callback_data="privacy:delete:confirm")],
        [InlineKeyboardButton(t(uid, "cancel"), callback_data="privacy:delete:cancel")]])
    await update.effective_message.reply_text(t(uid, "confirm_delete"), reply_markup=kb)

async def on_privacy_cb(update, context):
    q = update.callback_query
    await q.answer()
    uid = update.effective_user.id
    if q.data == "privacy:delete:confirm":
        ok = store.delete_user(pseudonymize(uid))
        context.user_data.clear()
        await q.edit_message_text(t(uid, "deleted" if ok else "delete_failed"))
    elif q.data == "privacy:delete:cancel":
        await q.edit_message_text(t(uid, "not_deleted"))
```

**Per-user localization**
```python
DEFAULT_LANG = os.getenv("DEFAULT_LANGUAGE", "en")
MESSAGES = {
    "en": {"welcome": "Welcome, {name}!", "slow_down": "Too many requests, try later."},
    "xx": {"welcome": "...", "slow_down": "..."},
}

def get_lang(user_id: int, tg_lang_code: str | None = None) -> str:
    lang = prefs.get(user_id, {}).get("language")
    if lang in MESSAGES:
        return lang
    if tg_lang_code and tg_lang_code[:2] in MESSAGES:
        return tg_lang_code[:2]
    return DEFAULT_LANG

def t(user_id: int, key: str, **kw) -> str:
    text = MESSAGES.get(get_lang(user_id), {}).get(key) or MESSAGES[DEFAULT_LANG].get(key) or key
    return text.format(**kw) if kw else text
```

**In-memory file upload with limits**
```python
import asyncio
MAX_FILE_SIZE = int(os.getenv("MAX_FILE_SIZE", 10 * 1024 * 1024))

async def on_document(update, context):
    doc = update.message.document
    uid = update.effective_user.id
    if not doc.file_name.lower().endswith(ALLOWED_EXT):
        return await update.message.reply_text(t(uid, "bad_type"))
    if doc.file_size and doc.file_size > MAX_FILE_SIZE:
        return await update.message.reply_text(t(uid, "too_large"))
    f = await context.bot.get_file(doc.file_id)
    data = bytes(await f.download_as_bytearray())
    result = await asyncio.to_thread(process_upload, data)
    await update.message.reply_text(render(result))
```

**Optional hardening (recommended, not yet implemented by any adopter)**
```python
import time
from collections import defaultdict, deque
from functools import wraps

ADMIN_IDS = {int(x) for x in os.getenv("ADMIN_USER_IDS", "").split(",") if x.strip()}

def admin_only(handler):
    @wraps(handler)
    async def wrapper(update, context, *a, **kw):
        user = update.effective_user
        if not user or user.id not in ADMIN_IDS:
            return
        return await handler(update, context, *a, **kw)
    return wrapper

def rate_limited(max_calls: int, per_seconds: float):
    buckets: dict[int, deque] = defaultdict(deque)
    def deco(handler):
        @wraps(handler)
        async def wrapper(update, context, *a, **kw):
            uid, now = update.effective_user.id, time.monotonic()
            dq = buckets[uid]
            while dq and now - dq[0] > per_seconds:
                dq.popleft()
            if len(dq) >= max_calls:
                if update.callback_query:
                    await update.callback_query.answer(t(uid, "slow_down"))
                elif update.effective_message:
                    await update.effective_message.reply_text(t(uid, "slow_down"))
                return
            dq.append(now)
            return await handler(update, context, *a, **kw)
        return wrapper
    return deco
```

**docker-compose.yml (polling, no ports)**
```yaml
services:
  bot:
    build: .
    restart: unless-stopped
    env_file: .env
    security_opt: ["no-new-privileges:true"]
    deploy:
      resources:
        limits: { cpus: "0.5", memory: 512M }
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
```

### Env vars
- `TELEGRAM_BOT_TOKEN`: required.
- `LOG_LEVEL`: optional.
- `MAX_FILE_SIZE`, `MAX_ROWS`: upload limits.
- `USER_ID_SALT`: HMAC salt for pseudonymous storage keys. Must be stable.
- `DATA_RETENTION_DAYS`: inactive-record cleanup (default 90).
- `FIREBASE_SERVICE_ACCOUNT_KEY`: path to a service-account JSON, if you use Firestore.
- `ADMIN_USER_IDS`: only if you add the admin decorator.

### Gotchas
- **Pattern-less callback handler.** A `CallbackQueryHandler` without `pattern=` catches *every* callback, so handlers registered after it in the same group never fire, and the button spinner hangs. Namespace your `callback_data` and give each handler a matching pattern.
- **Always `await query.answer()`**, even when there's nothing to say. Otherwise the client shows a loading spinner for about 15 seconds.
- **`update.message` is `None` in callback queries.** Reusing a command function from a button breaks if it calls `update.message.reply_text`. Use `update.effective_message` everywhere.
- **"Message is not modified"** `BadRequest` when re-rendering an unchanged keyboard. Catch it (or diff before editing).
- **`callback_data` is capped at 64 bytes.** Use short ids, not names.
- **Legacy `parse_mode='Markdown'` traps:**
  - `**bold**` isn't valid; legacy Markdown bold is `*bold*`.
  - Unescaped user input or exception text causes `can't parse entities` errors and leaks internals.
  - Prefer `ParseMode.HTML` with `html.escape`, or MarkdownV2 with `telegram.helpers.escape_markdown`.
- **Blocking I/O in async handlers.** Sync Firestore calls and pandas work stall the whole bot. Wrap them in `asyncio.to_thread`, or use async clients.
- **One poller per token.** A second instance (for example, a local dev run while prod is up) gets `Conflict: terminated by other getUpdates request`. Use a separate dev bot token.
- **Unstable salt.** If `USER_ID_SALT` falls back to a random per-process value, every restart orphans all pseudonymous records. Fail fast if it's unset.
- **Writes on every read.** Touching `last_accessed` on every preference read writes once per lookup, and the language is looked up many times per message. Cache per update, or throttle the touch.
- **Import-only Docker healthcheck.** `python -c "import telegram"` doesn't prove the bot is polling. Consider a heartbeat file touched by a `JobQueue` job.
- **Credential files in the image.** A service-account JSON in the build context gets baked in by `COPY . .` unless `.dockerignore` excludes `*.json` credential files. Mount it as a secret or volume instead.
- **Download size limit.** The standard Bot API limits bot file downloads to 20 MB. Check `document.file_size` before downloading, and keep the bytes in memory instead of writing to disk.
- **Webhooks vs polling.** Webhooks (HTTPS plus a secret token) only pay off at scale or on serverless. Polling on a small VM needs no ingress, TLS or domain.

### Playbook confidence: medium
Extracted from real local source (single implementation, not yet cross-checked against a second adopter). The admin-only and rate-limit decorators are recommended additions synthesized from the adopter's own security notes, not observed running code.

## Adoption
Used in **1** repo in this marketplace. It uses an async long-polling bot with inline-keyboard multi-select, opt-in consent and privacy commands backed by Firestore (with an in-memory fallback before consent), two-language per-user localization, in-memory CSV upload validation, and a single Docker Compose container on a small VM.
