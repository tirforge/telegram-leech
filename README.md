<div align="center">

# Telegram Leech / Mirror Bot

**A Telegram bot that mirrors files — direct links, torrents, Mega, YouTube — to Google Drive or Telegram, with per-user Drive support via OAuth.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/Thirupathi-pirate/telegram-leech/pulls)

</div>

A fork of [MirrorBot137](https://github.com/rahulkhatri137/mirrorbot137) (a feature-collection fork of [lzzy12/python-aria-mirror-bot](https://github.com/lzzy12/python-aria-mirror-bot)) that adds **per-user Google Drive uploads via OAuth** and **Cloudflare Tunnel support** for the OAuth callback server. Full credits to the original authors in [Credits](#credits).

> **Note:** This (or any custom) repo is not supported in the official bot support group. Use at your own risk.

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Bot Commands](#bot-commands)
- [Usage Examples](#usage-examples)
- [Per-User Google Drive (OAuth)](#per-user-google-drive-oauth)
- [Exposing the OAuth Server with Cloudflare Tunnel](#exposing-the-oauth-server-with-cloudflare-tunnel)
- [Deployment](#deployment)
- [Extras](#extras)
- [Contributing](#contributing)
- [Credits](#credits)
- [License](#license)

## Features

**Core mirror/leech**
- Mirror direct download links to Google Drive or Telegram
- Mirror Mega.nz links, Telegram files, and all yt-dlp supported links (YouTube, etc.)
- Mirror torrent files and magnet links (aria2)
- Mirror by reply; mirror/leech multiple links or files at once
- Custom filename support for direct links, Telegram files, and yt-dlp links
- Leech to Telegram with custom thumbnails, names, and Premium 4GB uploads
- Send files directly to the user's PM

**Drive**
- **Per-user Google Drive uploads via OAuth** (`/connect`, `/disconnect`) — each user uploads to their own Drive
- Service account support for cloning and uploading (rate-limit friendly)
- Upload to Team Drives; copy files between Drives (Autorclone)
- Multi-drive search, count drive files, index-link streaming (Bhadoo)

**File handling**
- Extract and upload archives: ZIP, RAR, TAR, 7z, ISO, WIM, CAB, GZIP, BZIP2, APM, ARJ, CHM, CPIO, CramFS, DEB, DMG, FAT, HFS, LZH, LZMA, LZMA2, MBR, MSI, MSLZ, NSIS, NTFS, RPM, SquashFS, UDF, VHD, XAR, Z
- Extract password-protected archives (you supply the password)

**Operations**
- Download/upload progress, speeds and ETAs; `/status` live tracking
- Speedtest, URL shortener support, custom buttons on file posts
- Self-update on startup and `/restart` via `UPSTREAM_REPO`
- Mirror log chat; Docker, Heroku, and restricted-host (HiddenCloud) deployments

## Tech Stack

- **Python 3.10–3.13**
- [python-telegram-bot 13](https://python-telegram-bot.org/) (command handlers) + [Pyrogram 2.0](https://docs.pyrogram.org/) (file transfers)
- [aria2](https://aria2.github.io/) — torrents/direct downloads
- [yt-dlp](https://github.com/yt-dlp/yt-dlp) — video/streaming links
- Google Drive API (`google-api-python-client`) + OAuth (`google-auth-oauthlib`)
- `aiohttp` — OAuth callback server
- PostgreSQL (`DATABASE_URL`, optional) — auth users, leech settings, RSS, pending tasks

## Quick Start

### Prerequisites

- Python 3.10+ and `pip`
- [aria2](https://aria2.github.io/) installed and running (or use `aria.sh`)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Telegram API ID/hash from https://my.telegram.org

### 1. Clone and install

```bash
git clone https://github.com/Thirupathi-pirate/telegram-leech.git
cd telegram-leech
pip install -r requirements.txt
```

### 2. Configure

```bash
cp config_sample.env config.env
```

Edit `config.env`: remove the `_____REMOVE_THIS_LINE_____=True` line and fill in the [required variables](#required-variables).

### 3. Start

```bash
./aria.sh            # start aria2 (download engine)
python3 main.py      # auto-installs missing deps, then starts the bot
```

or with Docker:

```bash
sudo docker build -t mirror-bot .
sudo docker run mirror-bot
```

### 4. Set commands with @BotFather

Paste the command list from the [Bot Commands](#bot-commands) section into BotFather → `/setcommands` so the bot shows a menu. Then message your bot and run `/mirror <link>`.

## Configuration

All settings live in `config.env` (see `config_sample.env`). Every value is also readable from environment variables.

### Required variables

| Variable | Description |
|---|---|
| `BOT_TOKEN` | Bot token from [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_API` | API ID from https://my.telegram.org (for Telegram file transfers) |
| `TELEGRAM_HASH` | API hash from https://my.telegram.org |
| `OWNER_ID` | Your Telegram user ID |
| `GDRIVE_FOLDER_ID` | Default Drive folder / team drive ID for uploads |
| `DOWNLOAD_DIR` | Local path where downloads are stored |
| `DOWNLOAD_STATUS_UPDATE_INTERVAL` | Seconds between status updates (recommended ≥ 10) |
| `AUTO_DELETE_MESSAGE_DURATION` | Seconds after which bot messages auto-delete (`-1` disables) |

### OAuth variables (per-user Drive)

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID` | OAuth web client ID from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | OAuth web client secret |
| `OAUTH_REDIRECT_URI` | Full callback URL, e.g. `http://localhost:8080/oauth/callback` or your tunnel URL |
| `OAUTH_SERVER_PORT` | Port for the callback server (default `8080`) |

### Optional variables

| Variable | Description |
|---|---|
| `UPSTREAM_REPO` / `UPSTREAM_BRANCH` | Self-update source repo/branch on restart |
| `DATABASE_URL` | PostgreSQL URL (auth/sudo users, leech settings, RSS, pending tasks) |
| `AUTHORIZED_CHATS` | Space-separated user/chat IDs allowed to use the bot |
| `SUDO_USERS` | Space-separated user IDs with sudo permissions |
| `IS_TEAM_DRIVE` | `True` if uploading to a Team Drive |
| `USE_SERVICE_ACCOUNTS` | `True` to use service accounts instead of per-user OAuth |
| `SESSION_STRING` | Pyrogram session string (downloads via your Telegram account) |
| `TG_SPLIT_SIZE` | Split size for Telegram uploads; empty = max (2GB / 4GB premium) |
| `AS_DOCUMENT` | Upload files to Telegram as documents instead of media |
| `CUSTOM_FILENAME` | Custom word appended to leeched file names |
| `CLONE_LIMIT` | Max clone size in GB |
| `STOP_DUPLICATE_CLONE` | Skip files already present in Drive (by name) |
| `RECURSIVE_SEARCH` | Enable recursive multi-drive search (`drive_folder` file) |
| `INDEX_URL` | Bhadoo/GoIndex index URL |
| `SHORTENER` / `SHORTENER_API` | URL shortener + API key |
| `CRYPT` | gdtot cookie for gdtot links |
| `MEGA_KEY` / `MEGA_USERNAME` / `MEGA_PASSWORD` | Mega.nz credentials |
| `UPTOBOX_TOKEN` | Uptobox token |
| `LOGS_CHATS` | Chat IDs where mirror logs are posted |
| `BOT_PM` | Send files to user PM |
| `BUTTON_THREE/FOUR/FIVE_NAME/URL` | Extra buttons under file posts |

## Bot Commands

Set these with BotFather:

```
mirror - Mirror
zipmirror - Mirror and upload as zip
unzipmirror - Mirror and extract files
leech - Leech to Telegram
zipleech - Leech and upload as zip
unzipleech - Leech and extract files
clone - Copy file/folder to Drive
count - Count file/folder of Drive
watch - Mirror yt-dlp supported link
zipwatch - Mirror yt-dlp supported link as zip
leechwatch - Leech through yt-dlp supported link
leechzipwatch - Leech yt-dlp support link as zip
leechset - Leech settings
setthumb - Set thumbnail
status - Get Mirror Status message
list - Search files in Drive
cancel - Cancel a task
cancelall - Cancel all tasks
del - Delete file/folder from Drive
log - Get the Bot Log
restart - Restart the Bot
stats - Bot Usage Stats
ping - Ping the Bot
connect - Connect your Google Drive
disconnect - Disconnect your Google Drive
help - All cmds with description
```

### Command reference

| Command | Description | Access |
|---|---|---|
| `/start`, `/help` | Start message, full command list | All |
| `/mirror <link>` | Mirror a link to Google Drive | Authorized |
| `/zipmirror <link>` | Mirror and upload as zip | Authorized |
| `/unzipmirror <link>` | Mirror and extract archives | Authorized |
| `/tarmirror <link>` | Mirror and upload as tar | Authorized |
| `/leech <link>` | Download and upload to Telegram | Authorized |
| `/zipleech <link>` | Leech as zip | Authorized |
| `/unzipleech <link>` | Leech and extract | Authorized |
| `/tarleech <link>` | Leech as tar | Authorized |
| `/watch <link> [quality] [\|name]` | Mirror yt-dlp links (YouTube, etc.) | Authorized |
| `/zipwatch` · `/tarwatch` | Watch + zip/tar | Authorized |
| `/leechwatch` · `/leechzipwatch` | Watch + leech to Telegram | Authorized |
| `/clone <drive-url>` | Copy file/folder between Drives | Authorized |
| `/count <drive-url>` | Count files/folders in Drive | Authorized |
| `/list <query>` | Search files in Drive | Authorized |
| `/status` | Show active task status | Authorized |
| `/speedtest` | Run a speed test | Authorized |
| `/leechset` | Leech settings (thumb, format) | Authorized |
| `/setthumb` | Set a custom thumbnail | Authorized |
| `/connect` | Connect your own Google Drive (OAuth) | Authorized |
| `/disconnect` | Remove your Drive credentials | Authorized |
| `/cancel` | Cancel one of your tasks | Task owner |
| `/cancelall` | Cancel all tasks | Sudo |
| `/del <drive-url>` | Delete file/folder from Drive | Sudo |
| `/log` | Get bot log file | Sudo |
| `/restart` | Restart the bot | Sudo |
| `/stats` | Show machine stats | Sudo |
| `/authorize` · `/unauthorize` · `/users` | Manage authorized users | Sudo |
| `/addsudo` · `/rmsudo` | Add/remove sudo users | Owner |

> Commands without a link argument show usage help. Reply-to-mirror also works: reply to a Telegram file with `/mirror`.

## Usage Examples

```bash
# Mirror a direct link to your Drive
/mirror https://example.com/file.zip

# Leech (download and send back to Telegram)
/leech https://example.com/video.mp4

# YouTube video in 720p
/watch https://www.youtube.com/watch?v=... 720

# Clone a public Drive folder
/clone https://drive.google.com/drive/folders/xxx

# Connect your personal Google Drive first
/connect
```

## Per-User Google Drive (OAuth)

By default (service accounts disabled), **every authorized user uploads to their own Google Drive** instead of the owner's `GDRIVE_FOLDER_ID`.

**Flow:**

1. User sends `/connect`.
2. Bot replies with a Google OAuth consent URL.
3. User authorizes in the browser; Google redirects to the bot's callback server.
4. Bot saves the credentials as `user_creds/<user_id>.json` and confirms in Telegram.
5. `/mirror`, `/clone`, etc. now upload to that user's Drive. `/disconnect` removes the credentials.

**How it works:**

- Auth precedence: `USE_SERVICE_ACCOUNTS=True` → service accounts; otherwise per-user OAuth.
- Credentials are stored per user with automatic token refresh (offline access).
- `user_creds/*.json` is gitignored (`*.json`); never commit credential files.
- Users who haven't run `/connect` get a clear "Use /connect first" error instead of silently uploading to someone else's Drive.

**Setting up OAuth (Google Cloud Console):**

1. Create a project and enable the **Google Drive API**.
2. **OAuth consent screen** → External → add your Google account as a test user (or publish the app).
3. **Credentials** → Create Credentials → **OAuth client ID** → **Web application**.
4. Add the callback URL as an authorized redirect URI:
   - Local: `http://localhost:8080/oauth/callback`
   - Tunnel: `https://<your-tunnel>.trycloudflare.com/oauth/callback`
5. Copy the Client ID and secret into `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`.

## Exposing the OAuth Server with Cloudflare Tunnel

The OAuth callback server listens on `0.0.0.0:<OAUTH_SERVER_PORT>` (default `8080`). If your host has no public IP, or you don't want to open ports, expose just this port with a Cloudflare Tunnel:

**Quick tunnel (no account, URL changes on restart):**

```bash
cloudflared tunnel --url http://localhost:8080
```

Copy the printed `https://<random>.trycloudflare.com` URL and set `OAUTH_REDIRECT_URI=https://<random>.trycloudflare.com/oauth/callback` **before** generating `/connect` links. Note: quick-tunnel URLs rotate each restart, so users must reconnect after a tunnel restart.

**Named tunnel (fixed hostname, recommended):**

```bash
cloudflared tunnel login
cloudflared tunnel create mirrorbot
cloudflared tunnel route dns mirrorbot oauth.example.com
cloudflared tunnel run mirrorbot --url http://localhost:8080
```

Then use `OAUTH_REDIRECT_URI=https://oauth.example.com/oauth/callback` and add that URL as an authorized redirect URI in Google Cloud Console.

> The tunnel only exposes the OAuth callback port. The bot itself talks to Telegram via long polling, so it needs no inbound ports.

## Deployment

### Docker

```bash
sudo docker build -t mirror-bot .
sudo docker run mirror-bot
```

### Heroku

- `TOKEN_PICKLE_URL` is required for Heroku.
- One-click: [![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://dashboard.heroku.com/new?button-url=https://github.com/rahulkhatri137/mirrorbot137-deployer&template=https://github.com/rahulkhatri137/mirrorbot137-deployer)
- GitHub Actions: set `HEROKU_API_KEY`, `HEROKU_APP_NAME`, `HEROKU_EMAIL`, `CONFIG_FILE_URL` in repo secrets, then run the **Deploy to Heroku** workflow manually.

### HiddenCloud / restricted hosts

`main.py` auto-installs `requirements.txt` before starting and adjusts `PYTHONPATH` for `.local` site-packages. A bundled `imghdr.py` shim covers Python 3.12/3.13 (module removed from stdlib). Start with `python3 main.py`.

## Extras

- **Service accounts** — set `USE_SERVICE_ACCOUNTS=True` and generate keys: `python3 gen_sa_accounts.py --quick-setup 1 --new-only`. Add them to a team drive with `python3 add_to_team_drive.py -d <TeamDriveID>`. One SA ≈ 750 GB/day; don't abuse.
- **Multi-drive search** — run `driveid.py` to generate the `drive_folder` file, set `RECURSIVE_SEARCH=True`.
- **YouTube authentication** — edit `.netrc` in the repo root: `machine host login username password my_youtube_password`.
- **gdtot cookies** — see the "Gdtot cookies" section of `config_sample.env` for the bookmarklet.

## Contributing

1. Fork the repo and create a branch: `git checkout -b feature/foo`
2. Commit your changes and push: `git push origin feature/foo`
3. Open a pull request against `master`.

Please run `python3 -m py_compile <changed files>` before opening the PR, and read `AGENTS.md` for architecture and security invariants (never commit `config.env`, `user_creds/`, or secrets).

## Credits

- **Per-user Google Drive OAuth** — OAuth server and per-user credential flow adapted from [telegram-drive-bot](https://github.com/telegram-drive-bot), rewritten for python-telegram-bot v13.
- **Original bot** — [Shivam Jha (lzzy12)](https://github.com/lzzy12) and [JaskaranSM](https://github.com/jaskaranSM) built [python-aria-mirror-bot](https://github.com/lzzy12/python-aria-mirror-bot) from scratch.
- **Fork base** — [MirrorBot137](https://github.com/rahulkhatri137/mirrorbot137) collected and merged community features.
- Feature contributions: [Sreeraj V R](https://github.com/SVR666/LoaderX-Bot), [SlamBot](https://github.com/breakdowns/slam-mirrorbot), [Ken](https://github.com/KenHV/Mirror-Bot), [Harsh](https://github.com/harshpreets63/Mirror-Bot).
- Service-account scripts: [AutoRclone](https://github.com/xyou365/AutoRclone).

## License

[GPL-3.0](LICENSE) — free to use, modify, and distribute; changes must remain open source.
