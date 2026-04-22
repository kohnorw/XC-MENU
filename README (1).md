# Dispatcharr User Manager

A self-hosted XC proxy that takes **one Dispatcharr XC user** and lets you create unlimited sub-users with expiry dates. Sub-users connect to this container using their own credentials — this tool proxies their requests upstream to Dispatcharr transparently.

---

## How it works

```
Sub-user (XC client)
      │  username/password + expiry checked here
      ▼
Dispatcharr User Manager  (this container, port 3000)
      │  always uses your one upstream XC account
      ▼
Dispatcharr  (your existing instance)
      │
      ▼
IPTV source
```

---

## Quick Start

### 1. Configure

Edit `docker-compose.yml` and set:

```yaml
UPSTREAM_URL: "http://192.168.1.100:9191"   # Your Dispatcharr URL
UPSTREAM_USERNAME: "your_xc_username"        # From Dispatcharr → Users tab
UPSTREAM_PASSWORD: "your_xc_password"        # The XC Password field in that user
ADMIN_USER: "admin"
ADMIN_PASS: "changeme"                       # Change this!
```

### 2. Run

```bash
docker compose up -d
```

### 3. Open admin UI

```
http://your-server-ip:3000/admin
```

Log in with your `ADMIN_USER` / `ADMIN_PASS`.

---

## Admin UI

- **Create users** — set username, password (or auto-generate), expiry date, max connections
- **View status** — Active / Expiring soon / Expired / Disabled badges
- **XC Info** — one-click copy of server URL, username, password, and M3U URL for each sub-user
- **Renew** — extend expiry by 7/14/30/60/90 days or a custom amount
- **Enable/Disable** — suspend a user without deleting them
- **Delete** — permanently removes the sub-user

---

## Sub-user XC connection details

Give each sub-user:

| Field    | Value                                      |
|----------|--------------------------------------------|
| URL      | `http://your-server-ip:3000`               |
| Username | their username                             |
| Password | their XC password                          |

---

## API endpoints (XC-compatible)

| Endpoint | Description |
|----------|-------------|
| `GET /player_api.php?username=X&password=Y` | Auth + user info |
| `GET /player_api.php?username=X&password=Y&action=get_live_streams` | Live streams |
| `GET /get.php?username=X&password=Y&type=m3u_plus` | M3U playlist |
| `GET /live/:user/:pass/:streamId.ts` | Stream proxy |

---

## Data persistence

User data is stored in `/data/users.db` (SQLite). The `./data` directory is mounted as a volume — back this up to preserve your users.

---

## Expiry enforcement

- Expired users are rejected **immediately** at the auth step
- A background cron job also marks them as inactive every hour
- Renewing a user re-activates them and extends from the current expiry date (or today if already expired)
