# XC-Menu

**XC-Menu** is a self-hosted IPTV sub-user manager and XC proxy. Feed it one upstream XC account and it lets you create multiple sub-users, each with their own credentials, expiry date, and connection limit. All streams are proxied transparently through your upstream account.

---

## How It Works

```
Your IPTV clients (XC app, Tivimate, etc.)
        │  authenticate with sub-user credentials
        ▼
   XC-Menu  (this container)
        │  validates expiry, connection limits
        │  proxies using your one upstream XC account
        ▼
  Dispatcharr / IPTV upstream
        │
        ▼
   IPTV source
```

---

## Getting Started

### 1. Extract and enter the folder

```bash
unzip xc-menu.zip
cd xc-menu
```

### 2. Start the container

```bash
docker-compose up -d
```

### 3. Open the setup wizard

Visit `http://your-server-ip:3000/admin` in your browser. On first run you will be taken through a two-step setup:

**Step 1 — Create your admin account**
Choose a username and password for the XC-Menu admin portal.

**Step 2 — Connect your IPTV server**
Enter your upstream server URL and XC credentials. XC-Menu will test the connection before saving.

Once setup is complete you will be taken straight to the dashboard.

---

## Rebuilding After an Update

```bash
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

---

## Admin Dashboard

### Users Table
Each sub-user shows:
- **Status badge** — Active, Expiring Soon, Expired, or Disabled
- **Expiry** — date and days remaining (or days overdue)
- **Max Connections** — how many simultaneous streams they can open

### Creating a User
Click **New User** and fill in:

| Field | Description |
|---|---|
| Username | The XC login username for this sub-user |
| Password | Leave blank to auto-generate a secure password |
| Display Name | Friendly label shown in the dashboard |
| Expiry | Pick a preset (24h Trial, 7/14/30/60/90 days, 1 year) or enter a custom date |
| Max Connections | Maximum simultaneous streams (default: 1) |
| Notes | Optional internal notes |

### User Actions

| Button | Description |
|---|---|
| **XC Menu** | Shows the sub-user's full connection info to copy and share |
| **Renew** | Extend expiry by a set number of days from current expiry or today |
| **Enable / Disable** | Suspend access without deleting the user |
| **Edit** | Change password, display name, expiry, connections, notes |
| **Delete** | Permanently remove the user |

### Live Streams Panel
Click the **Live Streams** stat card to expand the active connections panel. It shows:
- Which user is watching
- The channel name being streamed
- How long the stream has been running
- A **Kick** button to forcibly disconnect the stream

---

## Sub-User Connection Details

Give each sub-user the following to enter into their IPTV app:

| Field | Value |
|---|---|
| Server URL | `http://your-server-ip:3000` |
| Username | their username |
| Password | their XC password |

These can be copied directly from the **XC Menu** button on each user row.

---

## XC-Compatible Endpoints

| Endpoint | Description |
|---|---|
| `/player_api.php?username=X&password=Y` | Auth and user info |
| `/player_api.php?username=X&password=Y&action=get_live_streams` | Live stream list |
| `/player_api.php?username=X&password=Y&action=get_vod_streams` | VOD list |
| `/get.php?username=X&password=Y&type=m3u_plus` | M3U playlist |
| `/xmltv.php?username=X&password=Y` | EPG / TV guide |
| `/live/:user/:pass/:streamId.ts` | Live stream proxy |
| `/movie/:user/:pass/:file` | VOD proxy |
| `/series/:user/:pass/:file` | Series proxy |

---

## Expiry & Trial System

- **24h Trial** — sets expiry to exactly 24 hours from the moment of creation
- **Expiry is enforced immediately** — expired users are rejected at auth, not just on the next cron cycle
- **Hourly cron** — marks expired users as inactive in the background
- **Renewing** — extends from the current expiry date if it's in the future, or from today if already expired

---

## Data & Backups

All data is stored in a SQLite database at `/data/users.db` inside the container, mounted to `./data/users.db` on the host.

Back up this file to preserve all users, settings, and config:

```bash
cp ./data/users.db ./data/users.db.backup
```

---

## Security Notes

- Admin passwords are stored as SHA-256 hashes, never in plain text
- Sessions expire after 24 hours
- Sub-user credentials are validated on every stream request
- Max connection limits are enforced in real-time — exceeding the limit returns a 403

---

## Folder Structure

```
xc-menu/
├── docker-compose.yml   # Docker configuration
├── Dockerfile           # Container build file
├── package.json         # Node.js dependencies
├── src/
│   └── server.js        # Main application server
├── public/
│   └── admin/
│       ├── index.html   # Dashboard UI
│       ├── login.html   # Login page
│       └── setup.html   # First-run setup wizard
└── data/                # Persistent data volume
    └── users.db         # SQLite database (created on first run)
```
