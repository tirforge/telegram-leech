# Telegram Leech / Mirror Bot

A Python Telegram bot for mirroring files from the internet (direct links, torrents, Mega, YouTube/yt-dlp, Telegram files) to **Google Drive** or **Telegram** — with **per-user Google Drive support via OAuth**.

This is a fork of [MirrorBot137](https://github.com/rahulkhatri137/mirrorbot137), which is itself a feature-collection fork of [lzzy12/python-aria-mirror-bot](https://github.com/lzzy12/python-aria-mirror-bot). Full credits to the original authors at the bottom — all upstream features are preserved and extended.

> **Important**: This (or any custom) repo is not supported in the official bot support group. Use at your own risk.

## What's new in this fork

- **Per-user Google Drive support (OAuth)** — every authorized user can connect their *own* Google Drive with `/connect` and upload there, instead of everyone sharing the bot owner's folder. Credentials are stored per user (`user_creds/<user_id>.json`), auto-refreshed, and never shared.
- **Cloudflare Tunnel support** — expose the OAuth callback server publicly without a static IP / port-forwarding (see [Exposing the OAuth server with Cloudflare Tunnel](#exposing-the-oauth-server-with-cloudflare-tunnel)).
- **HiddenCloud / restricted-host compatibility fixes** — auto-install of requirements at startup, PYTHONPATH setup for `.local` site-packages, `imghdr` shim for Python 3.12/3.13, `urllib3<2` + `setuptools<71` pins, `psycopg2-binary` (no `pg_config` needed), and Telegraph API failure handling.
- **Pyrogram ↔ PTB conflict fixes** — `bot_token`/`SESSION_STRING` coexistence and `TG_SPLIT_SIZE` type-comparison fixes.

## Features

- Mirror direct download links to Google Drive / Telegram
- Mirror Mega.nz links to Google Drive / Telegram
- Mirror Telegram files to Google Drive
- Mirror all yt-dlp supported links (YouTube, etc.)
- Mirror torrent files and magnet links (aria2)
- Mirror by reply; mirror/leech multiple links or files
- Custom filename support (direct links, Telegram files, yt-dlp links)
- Extract and upload archives: ZIP, RAR, TAR, 7z, ISO, WIM, CAB, GZIP, BZIP2, APM, ARJ, CHM, CPIO, CramFS, DEB, DMG, FAT, HFS, LZH, LZMA, LZMA2, MBR, MSI, MSLZ, NSIS, NTFS, RPM, SquashFS, UDF, VHD, XAR, Z
- Copy files from someone's Drive to your Drive (Autorclone)
- Service account support for cloning and uploading
- Download/upload progress, speeds and ETAs
- Docker support
- Upload to Team Drives
- Index link support, URL shortener support, index-link streaming (Bhadoo)
- Leech to Telegram with custom thumbnail and names
- Multi-drive search, speedtest, count drive files
- Extract password-protected archives (you supply the password)
- Self-update on startup and `/restart` via `UPSTREAM_REPO`
- Mirror log chat; Telegram Premium 4GB uploads; send files directly to PM
- **Per-user Google Drive uploads via OAuth (`/connect`)**

## How to use

### 1. Deploy

The bot runs as a Python process (Docker recommended). See [Deployment](#deployment) for Docker, Heroku, and GitHub Actions options.

### 2. Configure

Copy the sample config and fill it in:

```bash
cp config_sample.env config.env
```

Remove the `_____REMOVE_THIS_LINE_____=True` line, then fill at minimum:

| Var | Purpose |
|---|---|
| `BOT_TOKEN` | Token from [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_API`, `TELEGRAM_HASH` | From https://my.telegram.org (for Telegram file transfers) |
| `OWNER_ID` | Your Telegram user ID |
| `GDRIVE_FOLDER_ID` | Default Drive folder/team drive ID for uploads |
| `DOWNLOAD_DIR` | Local download path |

See the full [Configuration reference](#configuration-reference) below.

### 3. Start the bot

```bash
sudo dockerd   # if not already running
sudo docker build . -t mirror-bot
sudo docker run mirror-bot
```

or locally:

```bash
pip install -r requirements.txt
python3 main.py   # auto-installs missing requirements, then starts the bot
```

### 4. Set commands with @BotFather

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
del - Delete file/folder from Drive{owner}
log - Get the Bot Log{owner}
restart - Restart the Bot{sudo}
stats - Bot Usage Stats
ping - Ping the Bot
connect - Connect your Google Drive{authorized}
disconnect - Disconnect your Google Drive{authorized}
help - All cmds with description
```

## Per-user Google Drive (OAuth) — how it works

By default (without service accounts), **every authorized user uploads to their own Google Drive** instead of the owner's `GDRIVE_FOLDER_ID`.

Flow:

1. The user sends `/connect` to the bot.
2. The bot replies with a Google OAuth consent URL.
3. The user authorizes in the browser; Google redirects to the bot's OAuth callback server.
4. The bot saves the credentials as `user_creds/<user_id>.json` and confirms in Telegram.
5. From then on, `/mirror`, `/clone`, etc. upload to that user's Drive. `/disconnect` removes the credentials.

Details:

- Auth precedence: if `USE_SERVICE_ACCOUNTS=True`, service accounts are used as before. Otherwise each user's own OAuth credentials are used.
- Credentials are stored per user, token expiry is handled automatically (offline refresh token).
- `user_creds/*.json` is gitignored (`*.json`); never commit credential files.
- Users who haven't run `/connect` yet get a clear "Use /connect first" error instead of silently uploading to someone else's Drive.

### OAuth configuration

In the [Google Cloud Console](https://console.cloud.google.com/apis/credentials):

1. Create a project, enable the **Google Drive API**.
2. **OAuth consent screen** → External → add your Google account as a test user (or publish the app).
3. **Credentials** → Create Credentials → **OAuth client ID** → **Web application**.
4. Add your callback URL as an authorized redirect URI:
   - Local: `http://localhost:8080/oauth/callback`
   - Cloudflare Tunnel: `https://<your-tunnel>.trycloudflare.com/oauth/callback` (or your fixed tunnel hostname)
5. Copy the **Client ID** and **Client secret** into config:

```
GOOGLE_CLIENT_ID=xxxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxx
OAUTH_REDIRECT_URI=https://<your-host>/oauth/callback
OAUTH_SERVER_PORT=8080
```

The callback endpoint is `GET /oauth/callback?code=...&state=<user_id>`, served by `oauth_server.py` on `OAUTH_SERVER_PORT`.

### Exposing the OAuth server with Cloudflare Tunnel

The OAuth callback server listens on `0.0.0.0:<OAUTH_SERVER_PORT>` (default `8080`). If your host has no public IP or you don't want to open ports, expose it with a Cloudflare Tunnel:

**Quick tunnel (no account needed, URL changes on restart):**

```bash
cloudflared tunnel --url http://localhost:8080
```

Copy the `https://<random>.trycloudflare.com` URL it prints and set `OAUTH_REDIRECT_URI=https://<random>.trycloudflare.com/oauth/callback` **before** generating `/connect` links. Note: quick-tunnel URLs rotate each restart, so regenerate the config and ask users to reconnect if the tunnel restarts.

**Named tunnel (fixed hostname, recommended):**

```bash
cloudflared tunnel login
cloudflared tunnel create mirrorbot
cloudflared tunnel route dns mirrorbot oauth.example.com
cloudflared tunnel run mirrorbot --url http://localhost:8080
```

Then use `OAUTH_REDIRECT_URI=https://oauth.example.com/oauth/callback` and add `https://oauth.example.com/oauth/callback` as an authorized redirect URI in Google Cloud Console.

> The tunnel only exposes the OAuth callback port — it does not expose the Telegram bot itself (Telegram connects to the bot via long polling).

## Configuration reference

### Required fields

- `BOT_TOKEN`: Telegram bot token from [@BotFather](https://t.me/BotFather)
- `TELEGRAM_API` / `TELEGRAM_HASH`: from https://my.telegram.org — authenticates the account used to download/upload Telegram files. `Int`
- `OWNER_ID`: Telegram user ID of the bot owner. `Int`
- `GDRIVE_FOLDER_ID`: Folder/TeamDrive ID where mirrors upload by default (used when service accounts are enabled or by the owner).
- `DOWNLOAD_DIR`: Local path where downloads are stored.
- `DOWNLOAD_STATUS_UPDATE_INTERVAL`: Seconds between progress updates (recommended ≥ `10`). `Int`
- `AUTO_DELETE_MESSAGE_DURATION`: Seconds after which the bot deletes its messages; `-1` disables. `Int`

### OAuth (per-user Drive) fields

- `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`: OAuth web client from Google Cloud Console.
- `OAUTH_REDIRECT_URI`: Full callback URL, e.g. `http://localhost:8080/oauth/callback` or `https://<tunnel>/oauth/callback`.
- `OAUTH_SERVER_PORT`: Port for the callback server. Default `8080`. `Int`

### Optional fields

- `UPSTREAM_REPO` / `UPSTREAM_BRANCH`: self-update source repo/branch on restart. For private repos use `https://username:{token}@github.com/...`. Default branch `master`.
- `ACCOUNTS_ZIP_URL`: Direct link to a ZIP of service accounts to load externally.
- `TOKEN_PICKLE_URL`: Direct link to load `token.pickle` externally (required for Heroku deploys).
- `DATABASE_URL`: Postgres URL — stores auth/sudo users, per-user leech settings + thumbnails, RSS data, incomplete tasks. On Heroku, delete from `config.env` and let Heroku inject it.
- `AUTHORIZED_CHATS`: user/chat ids (space-separated) allowed to use the bot.
- `SUDO_USERS`: user ids with sudo permissions (space-separated).
- `IS_TEAM_DRIVE`: `True` if uploading to a Team Drive. `Bool`
- `USE_SERVICE_ACCOUNTS`: use service accounts instead of per-user OAuth. `Bool` — see [Service accounts](#using-service-accounts)
- `INDEX_URL`: Bhadoo/GoIndex index URL for link previews.
- `MEGA_KEY` / `MEGA_USERNAME` / `MEGA_PASSWORD`: Mega.nz API key / premium credentials.
- `UPTOBOX_TOKEN`: token for uptobox mirrors.
- `STOP_DUPLICATE_CLONE`: skip files already present in Drive (by filename). `Bool`
- `CLONE_LIMIT`: max clone size in GB. `Int`
- `VIEW_LINK`: use `?a=view` index links (Bhadoo-compatible). `Bool`
- `IGNORE_PENDING_REQUESTS`: ignore pending requests after restart. `Bool`
- `SESSION_STRING`: Pyrogram session string for downloading/uploading via your Telegram account (`python3 generate_string_session.py`). Note: with a session string the bot works in groups/channels, not PM.
- `TG_SPLIT_SIZE`: split size in bytes for Telegram uploads; empty = max (2GB, 4GB for premium).
- `AS_DOCUMENT`: upload Telegram files as documents instead of media. `Bool`
- `CUSTOM_FILENAME`: prefix/suffix added to leeched filenames.
- `SHORTENER` / `SHORTENER_API`: URL shortener + API key (exe.io, gplinks.in, shrinkme.io, urlshortx.com, shortzon.com).
- `CRYPT`: gdtot cookie for gdtot link generation (see [Gdtot cookies](#gdtot-cookies)).
- `RECURSIVE_SEARCH`: recursive multi-drive search using the `drive_folder` file.
- `LOGS_CHATS`: chat ids where mirror logs are posted (bot must be admin).
- `BOT_PM`: send files directly to user PM. `Bool`
- `BUTTON_THREE_NAME/URL`, `BUTTON_FOUR_NAME/URL`, `BUTTON_FIVE_NAME/URL`: extra buttons under file posts.

Concurrent download limit: edit `MAX_CONCURRENT_DOWNLOADS` in `aria.sh`.

## Deployment

### Docker

```bash
sudo dockerd
sudo docker build . -t mirror-bot
sudo docker run mirror-bot
```

### Heroku

- `TOKEN_PICKLE_URL` is required for Heroku.
- One-click: [![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://dashboard.heroku.com/new?button-url=https://github.com/rahulkhatri137/mirrorbot137-deployer&template=https://github.com/rahulkhatri137/mirrorbot137-deployer)
- GitHub Actions workflow (manual trigger): set `HEROKU_API_KEY`, `HEROKU_APP_NAME`, `HEROKU_EMAIL`, `CONFIG_FILE_URL` in repo secrets, then run **Deploy to Heroku** in Actions.

### HiddenCloud / restricted hosts

`main.py` auto-installs `requirements.txt` before starting the bot and adjusts `PYTHONPATH` for `.local` site-packages. A bundled `imghdr` shim covers Python 3.12/3.13 where the module was removed from the stdlib.

## Multi search IDs

To list from multiple team drives/folders, run `driveid.py` to generate `drive_folder`, or create it manually:

```
MyTdName tdID IndexLink(if available)
MyTdName2 tdID IndexLink(if available)
```

Set `RECURSIVE_SEARCH=True`.

## Getting Google OAuth API credentials

- [Google Cloud Console](https://console.developers.google.com/apis/credentials) → OAuth Consent tab → fill and save.
- Credentials tab → Create Credentials → OAuth Client ID → **Web application** (this fork uses the web OAuth flow for per-user uploads).
- Enable the **Google Drive API**.
- For the legacy owner token path: download `credentials.json` to the repo root, then run:

```
pip install google-api-python-client google-auth-httplib2
python3 generate_drive_token.py
```

## Gdtot cookies

1. Log in at [gdtot](https://new.gdtot.top).
2. Run the bookmarklet below in the browser address bar (prepend `javascript:` if missing), copy the alert value, and paste it into `CRYPT` in `config.env`:

```js
javascript:(function () {
  const input = document.createElement('input');
  COOKIE = JSON.parse(JSON.stringify({cookie: document.cookie}));
  input.value = COOKIE['cookie'].split('crypt=')[1];
  document.body.appendChild(input);
  input.focus();
  input.select();
  document.execCommand('copy');
  document.body.removeChild(input);
})();
```

## Generate database

**ElephantSQL (free):**
1. Create an account at https://elephantsql.com
2. Create New Instance → Select Region → Review → Create instance
3. Copy the URL into `DATABASE_URL`.

**Heroku PostgreSQL:**
<p><a href="https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1"><img src="https://img.shields.io/badge/See%20Dev.to-black?style=for-the-badge&logo=dev.to" width="160""/></a></p>

## Using service accounts

Set `USE_SERVICE_ACCOUNTS=True` — recommended when uploading to a team drive, or when you prefer the classic single-owner mode. Scripts thanks to [AutoRclone](https://github.com/xyou365/AutoRclone). 1 service account can copy ~750 GB/day; 1 project can hold 100 accounts (~75 TB/day). Don't abuse it.

```bash
python3 gen_sa_accounts.py --quick-setup 1 --new-only
# re-download existing keys:
python3 gen_sa_accounts.py --download-keys project_id
```

### Add service accounts to a team drive

```bash
python3 add_to_team_drive.py -d SharedTeamDriveSrcID
```

## Youtube-dl authentication

Edit the `.netrc` file in the repo root:

```
machine host login username password my_youtube_password
```

One line per extractor host (youtube, twitch, ...).

## Credits

- **Per-user Google Drive OAuth** — OAuth server and per-user credential flow adapted from [telegram-drive-bot](https://github.com/telegram-drive-bot) (per-user Google Drive OAuth system), rewritten for python-telegram-bot v13 and this codebase.
- **Base bot** — [Shivam Jha (lzzy12)](https://github.com/lzzy12) and [JaskaranSM (Zero Cool)](https://github.com/jaskaranSM) built the original [python-aria-mirror-bot](https://github.com/lzzy12/python-aria-mirror-bot) from scratch.
- **Fork** — [MirrorBot137](https://github.com/rahulkhatri137/mirrorbot137) collected and merged features from many community forks; this repository builds on it.
- [Sreeraj V R](https://github.com/SVR666/LoaderX-Bot), [SlamBot](https://github.com/breakdowns/slam-mirrorbot), [Ken](https://github.com/KenHV/Mirror-Bot), [Harsh](https://github.com/harshpreets63/Mirror-Bot) for features and improvements.
- [AutoRclone](https://github.com/xyou365/AutoRclone) for the service-account scripts.

## License

GPL-3.0 — see [LICENSE](LICENSE).
