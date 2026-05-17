# XC-Menu

**XC-Menu** is a self-hosted IPTV sub-user manager and XtreamCodes proxy. You feed it one upstream XC account and it lets you create multiple sub-users, each with their own credentials, expiry date, and connection limit. All streams are proxied transparently through your upstream account — your clients never touch the upstream server directly.

---

## How It Works

```
Your IPTV clients (TiviMate, IPTV Smarters, VLC, etc.)
        │  authenticate with sub-user credentials
        ▼
   XC-Menu  (this container)
        │  validates credentials, expiry, connection limits
        │  proxies streams using your upstream XC account
        ▼
  Your upstream IPTV / Dispatcharr server
```

---

## Quick Start

### 1. Deploy with Docker Compose

```bash
docker compose up -d
```

### 2. Open the setup wizard

Visit `http://your-server-ip:3000/admin` in a browser. First run takes you through a two-step wizard:

**Step 1 — Create your admin account**
Choose a username and password for the XC-Menu admin portal.

**Step 2 — Connect your upstream server**
Enter your upstream IPTV server URL and XC credentials. XC-Menu tests the connection before saving.

Once complete you land on the dashboard automatically.

---

## Updating

```bash
# Upload and extract the new release (no subfolder created)
scp xc-menu-updated.tar.gz user@yourserver:/opt/xc-menu/
cd /opt/xc-menu
tar -xzf xc-menu-updated.tar.gz

# Rebuild and restart — your database and config are preserved
docker compose down
docker compose up -d --build
```

Your data lives in the `users.db` volume and is never touched by an update.

---

## Admin Dashboard

### Stats Row

Four cards at the top give an at-a-glance view:

| Card | Description |
|---|---|
| **Total Users** | All sub-users ever created |
| **Active** | Users with valid, non-expired accounts |
| **Expired** | Users whose expiry date has passed |
| **Live Streams** | Number of streams open right now — click to expand the panel |

---

### Active Streams Panel

Click the **Live Streams** card to open the active streams panel. It shows every stream currently being proxied:

| Column | Description |
|---|---|
| User | Which sub-user opened the stream |
| Channel | Channel name and stream ID |
| Started | Time the stream was opened |
| Duration | How long it has been running |

#### Kill Switch

A hard kill switch sits at the top of the panel. When activated:

- All active streams are **immediately terminated** — the TCP socket to the IPTV player is destroyed, not gracefully closed, so the player drops within seconds
- New stream connections are **blocked** at the server until the switch is turned off
- A confirmation dialog is shown before activating to prevent accidents

Toggle it off to restore streaming for all users.

#### Per-Stream Kill Button

Each row has a **Kill** button that terminates that individual stream. It:

- Destroys the upstream connection to your IPTV provider
- Force-closes the TCP socket to the IPTV player
- Fades the row out of the panel immediately

---

### Sub-Users Table

#### Creating a User

Click **New User** and fill in:

| Field | Description |
|---|---|
| Username | The XC login username for this sub-user |
| Password | Leave blank to auto-generate a secure 12-character password |
| Display Name | Friendly label shown in the dashboard |
| Expiry | Pick a preset or enter a custom date |
| Max Connections | Maximum simultaneous streams (default: 1) |
| Notes | Optional internal notes |

**Expiry presets:** 24h Trial · 7 days · 14 days · 30 days · 60 days · 90 days · 1 year

#### User Actions

| Button | Description |
|---|---|
| **XC Info** | Shows the sub-user's full connection details to copy and share |
| **Renew** | Extend expiry by 7, 14, 30, 60, or 90 days from current expiry (or today if already expired) |
| **Enable / Disable** | Suspend access without deleting the user |
| **Edit** | Change password, display name, expiry, connection limit, or notes |
| **Delete** | Permanently remove the user |

---

## Giving Users Their Connection Details

Click **XC Info** on any user row. Share these with them:

| Field | Value |
|---|---|
| Server URL | `http://your-server-ip:3000` |
| Username | their username |
| Password | their password |
| M3U URL | `http://your-server-ip:3000/get.php?username=X&password=Y&type=m3u_plus` |
| EPG URL | `http://your-server-ip:3000/xmltv.php?username=X&password=Y` |

---

## XC-Compatible Endpoints

XC-Menu is fully compatible with any app that supports XtreamCodes:

| Endpoint | Description |
|---|---|
| `/player_api.php?username=X&password=Y` | Auth check and user info |
| `/player_api.php?...&action=get_live_streams` | Live channel list (served from cache) |
| `/player_api.php?...&action=get_live_categories` | Live categories (served from cache) |
| `/player_api.php?...&action=get_vod_streams` | VOD list (served from cache) |
| `/player_api.php?...&action=get_vod_categories` | VOD categories (served from cache) |
| `/player_api.php?...&action=get_series` | Series list (served from cache) |
| `/get.php?username=X&password=Y&type=m3u_plus` | M3U playlist |
| `/xmltv.php?username=X&password=Y` | EPG / TV guide (proxied from upstream) |
| `/live/:user/:pass/:streamId.ts` | Live stream proxy |
| `/movie/:user/:pass/:file` | VOD proxy |
| `/series/:user/:pass/:file` | Series proxy |
| `/timeshift/:user/:pass/*` | Timeshift proxy |

---

## Stream & Connection Management

### Connection Limits

Max connections are enforced in real-time on every stream request. If a user already has the maximum number of streams open, new requests return `403 Max connections reached`.

### Ghost Stream Reaper

A background reaper runs every 2 minutes and sweeps the active connections map for streams whose underlying TCP socket is already dead (destroyed, dropped, or no longer writable). Any ghost connections found are evicted automatically. This prevents stale entries from blocking users due to hit max-connection limits.

### Stream Lifecycle

Connections are cleaned up on any of the following events:

- Client disconnects (player closed, network drop)
- Upstream ends the stream (provider-side close)
- Admin kills the stream (Kill button or Kill Switch)
- Ghost reaper detects a dead socket

---

## Channel Cache

On startup (and every 6 hours), XC-Menu fetches and caches all live streams, VOD, series, and their categories from your upstream server. Playlist and category requests are served from this cache, which means:

- Faster response times for your clients
- No upstream timeout risk on large channel lists
- Your upstream credentials are never exposed to sub-users

The cache refreshes automatically. To force a refresh, restart the container.

---

## Expiry System

- **Expiry is enforced at auth time** — expired users are rejected immediately, not just on the next cron cycle
- **Hourly cron** — marks any newly expired users as inactive in the database
- **Trial accounts** — the 24h preset sets expiry to exactly 24 hours from the moment of creation
- **Renewing** — extends from the current expiry if it's still in the future, or from today if already expired

---

## Data & Backups

All data is stored in a SQLite database mounted at `/data/users.db` inside the container. Back it up with:

```bash
cp ./data/users.db ./data/users.db.backup
```

This single file contains all users, settings, admin credentials, and config.

---

## Security

- Admin passwords are stored as SHA-256 hashes — never plain text
- Admin sessions expire after 24 hours
- Sub-user credentials are validated on every single stream and API request
- The upstream server URL and credentials are never exposed to sub-users
- The kill switch state persists in the database across restarts

---

## Folder Structure

```
xc-menu/
├── docker-compose.yml      # Docker Compose configuration
├── Dockerfile              # Container build file
├── package.json            # Node.js dependencies
├── src/
│   └── server.js           # Express server — proxy, API, stream management
└── public/
    └── admin/
        ├── index.html      # Main dashboard UI
        ├── login.html      # Admin login page
        └── setup.html      # First-run setup wizard
```

**Runtime data** (created automatically, not in the repo):

```
/data/
└── users.db                # SQLite database — all users, config, and settings
```
