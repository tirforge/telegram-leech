# AGENTS.md

This file is the entry point for AI agents (and humans) entering this repository. Read it first to understand the architecture, the rules that must not be broken, and how to verify changes.

## What this is

A Python Telegram **mirror / leech bot**: users send links (direct, torrent/magnet, Mega, yt-dlp, Telegram files) and the bot downloads and uploads them to **Google Drive** or back to **Telegram**.

- Fork of `rahulkhatri137/mirrorbot137` (itself a fork of `lzzy12/python-aria-mirror-bot`).
- Key fork feature: **per-user Google Drive uploads via OAuth** (`/connect`, `/disconnect`).
- Targets: Docker, Heroku, and restricted hosts (HiddenCloud/HF Spaces) via `main.py`.

## Runtime stack

- Python 3.10–3.13 (3.12/3.13 need the bundled `imghdr.py` shim and monkey-patches).
- Two Telegram clients run side by side:
  - **python-telegram-bot v13** (`telegram.ext`) — command handlers, `bot/__init__.py` builds `updater`/`dispatcher`.
  - **Pyrogram** (`bot/__main__.py` + `bot/__init__.py`) — file transfer, premium detection.
- Download engines: aria2 (torrents/direct), yt-dlp, Mega (via megasdkrest), Telegram downloader.
- `google-api-python-client` for Drive; `aiohttp` for the OAuth callback server.

## Repo structure

```
main.py                  # HiddenCloud entry: auto-installs requirements, starts bot
oauth_server.py          # Per-user Google Drive OAuth (aiohttp callback + /connect, /disconnect)
config_sample.env        # All env vars (copy to config.env; NEVER commit config.env)
bot/__init__.py          # Config loading, Telegram clients, command registration
bot/__main__.py          # Entry point
bot/modules/             # Command modules (mirror, leech, clone, list, authorize, ...)
bot/helper/mirror_utils/ # downloaders, uploaders (gdriveTools.py), status helpers
bot/helper/telegram_helper/  # commands, filters, message utils
bot/helper/ext_utils/    # db_handler, bot_utils, fs_utils, telegraph_helper
generate_drive_token.py  # legacy owner token.pickle generator (not the OAuth path)
gen_sa_accounts.py       # service-account key generator
add_to_team_drive.py     # add SAs to a team drive
driveid.py               # generate drive_folder for multi-drive search
aria.sh / aria.bat       # start aria2c (set MAX_CONCURRENT_DOWNLOADS here)
```

## Auth model (read before touching upload code)

Auth precedence in `gdriveTools.py` → `authorize()`:

1. `USE_SERVICE_ACCOUNTS=True` → rotate a random service account from `accounts/`.
2. Otherwise → **per-user OAuth**: `oauth_server.get_user_service(user_id)`, loading `user_creds/<user_id>.json`, refreshing if expired, erroring with "Use /connect first" if missing.

Rules:

- `user_id` must flow from the command (e.g. `/mirror`) → `MirrorListener` → `GoogleDriveHelper` (`bot/modules/mirror.py`). Never drop it.
- New upload paths must keep this precedence. Don't reintroduce a global default that bypasses per-user creds when service accounts are off.
- OAuth env vars: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `OAUTH_REDIRECT_URI`, `OAUTH_SERVER_PORT` (default 8080). Callback route: `GET /oauth/callback?code=...&state=<user_id>` (`oauth_server.py`).

## Security invariants — never break these

- **Never commit** `config.env`, `user_creds/*.json`, `token.pickle`, `accounts/*.json`, `.netrc` contents, `*.session`, or any secrets. `.gitignore` already covers `*.json`/`*.pickle`/`config.env` — do not weaken it.
- `AUTHORIZED_CHATS` / `SUDO_USERS` gating in `bot/helper/telegram_helper/filters.py` must stay in front of owner/sudo commands (`del`, `log`, `restart`, `stats`, `connect`, `disconnect`).
- OAuth `state` parameter is the Telegram `user_id` — it must be validated (`int(state)`) in the callback before saving credentials.

## Gotchas

- **No root `pnpm`/scripts** — this is not the Open Design monorepo; run Python directly (`python3 main.py`, `pip install -r requirements.txt`).
- **Python 3.12/3.13**: `imghdr` was removed from stdlib — the local `imghdr.py` shim at the repo root must stay until upstream deps drop it. `urlparse`/`urlsplit` monkey-patches in `bot/__init__.py` handle `lk21`/`cloudscraper` ValueError crashes.
- **PTB v13 quirks**: `CommandHandler` + `run_async=True`; do not migrate to PTB v20+ without reworking every handler and the `Updater` API.
- **Pyrogram vs PTB**: both may try to use `bot_token` — `SESSION_STRING` takes precedence for the Pyrogram client; the conflict fix lives in `bot/__init__.py` and `bot/__main__.py`.
- **Self-update**: `UPSTREAM_REPO`/`UPSTREAM_BRANCH` pulls this repo at startup/`/restart`. Don't commit changes that break an existing deployment's `config.env` contract (new required vars must get defaults or be optional).
- Requirements pins are deliberate: `google-auth-oauthlib==0.4.6`, `urllib3<2`, `setuptools<71`, `psycopg2-binary`. Verify a pin's reason before bumping.

## Verification

- Syntax check changed Python files:
  ```bash
  python3 -m py_compile bot/**/*.py bot/__init__.py bot/__main__.py oauth_server.py main.py 2>/dev/null || python3 -m py_compile <changed files>
  ```
- Import smoke test (no network): `python3 -c "import bot"` may fail on missing env; prefer `py_compile` plus a targeted review of the changed module.
- After editing `gdriveTools.py` or `oauth_server.py`, re-check both auth branches (service accounts vs per-user) and that `user_id` propagates.
- There is no test suite in this repo — manual verification against a test bot token is the only runtime check. State that clearly when you change runtime behavior.
- Before committing, run `git diff` and confirm no secrets, no `config.env`, no `user_creds/`.

## Style

- Plain Python 3, 4-space indent, snake_case. No type-checking framework in use.
- Keep AI-generated code free of watermark comments (e.g. stray two-letter `#XX` markers) — they were cleaned from this repo and must not return.
