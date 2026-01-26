# Radicale CalDAV Server

Self-hosted CalDAV server for Ursine project at `https://caldav.ber.computer`.

## Overview

- **Server**: Radicale 3.5.3
- **URL**: https://caldav.ber.computer
- **Purpose**: Calendar storage for Claude Code skills
- **Authentication**: Required (username/password)

## Installation

**Tested on:** Debian 13.x (trixie)

```bash
apt install radicale python3-passlib python3-bcrypt apache2-utils
pip install caldav
```

## Configuration

### Radicale Config: `/etc/radicale/config`

```ini
[server]
hosts = 127.0.0.1:5232

[storage]
filesystem_folder = /var/lib/radicale/collections

[auth]
type = htpasswd
htpasswd_filename = /etc/radicale/users
htpasswd_encryption = bcrypt

[rights]
type = owner_only
file = /etc/radicale/rights

[web]
type = none
```

### Authentication: `/etc/radicale/users`

Uses bcrypt hashed passwords stored in htpasswd format.

**Format:** `username:<bcrypt_hash>`

**To update password:**
```bash
htpasswd -nbB username "new_password" > /etc/radicale/users
chown radicale:radicale /etc/radicale/users
chmod 640 /etc/radicale/users
systemctl restart radicale
```

**Credentials stored in:**
- `/etc/radicale/users` - bcrypt hash (for Radicale authentication)
- `~/.claude/skills/cal-add/.env` - for /cal-add skill
- `~/.bashrc` as `CALDAV_USER` and `CALDAV_PASS` - for /cal skills

### Web Server: nginx + Let's Encrypt

**Why Let's Encrypt instead of Cloudflare proxy?**

CalDAV protocol requires direct server-to-client communication for proper WebDAV operations. Cloudflare's proxy doesn't fully support the CalDAV/WebDAV protocol extensions (PROPFIND, REPORT, MKCALENDAR, etc.), causing sync failures and authentication issues. Let's Encrypt provides direct SSL/TLS termination while maintaining full protocol compatibility.

**nginx configuration: `/etc/nginx/sites-available/caldav.ber.computer`**

```nginx
server {
    listen 80;
    server_name caldav.ber.computer;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name caldav.ber.computer;
    http2 on;

    ssl_certificate /etc/letsencrypt/live/caldav.ber.computer/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/caldav.ber.computer/privkey.pem;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;

    client_max_body_size 100M;
    add_header DAV '1, 2, 3, calendar-access' always;

    # Simple redirect for iCal feed
    location = /calendar.ics {
        return 301 /ursine/<calendar-uuid>/;
    }

    location / {
        proxy_pass http://127.0.0.1:5232;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Remote-User $remote_user;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        proxy_buffering off;
        proxy_request_buffering off;
    }

    access_log /var/log/nginx/caldav.ber.computer-access.log;
    error_log /var/log/nginx/caldav.ber.computer-error.log;
}
```

**DNS Setup:**
- Create an A record pointing to your server IP
- **Important**: Set Cloudflare proxy to "DNS only" (gray cloud icon)
- CalDAV will not work through Cloudflare's orange cloud proxy

**SSL Certificate:**
```bash
# Obtain certificate (Cloudflare proxy must be disabled temporarily)
certbot certonly --nginx -d caldav.ber.computer

# Certificates auto-renew via certbot timer
# Manual renewal: certbot renew --nginx
```

## Calendar URLs

**Calendar UUID:** `<uuid>`

- **Full path:** `https://caldav.ber.computer/ursine/<uuid>/`
- **Simplified:** `https://caldav.ber.computer/ursine/`
- **Shortcut:** `https://caldav.ber.computer/calendar.ics` (redirects to full path)

All URLs require authentication.

## Services

```bash
# Radicale status
systemctl status radicale

# nginx status
systemctl status nginx

# Radicale logs
journalctl -u radicale -f
```

## Claude Code Skills

This CalDAV server is used by four Claude Code skills for calendar management:

### `/cal` Skill - Read Calendar Events

A Claude Code skill that reads and displays events from calendars. Supports authenticated CalDAV URLs and public iCal/webcal feeds.

**Usage:**
```bash
# Default Ursine calendar (uses env vars for auth)
/cal

# Specific calendar URL
/cal "https://caldav.ber.computer/ursine/"

# Other public calendars (iCloud, etc.)
/cal "webcal://p157-caldav.icloud.com/published/2/..."
```

**Configuration:**
- Uses `CALDAV_USER` and `CALDAV_PASS` environment variables for authenticated calendars
- Default calendar configured in `~/.claude/skills/cal/.env`
- Supports `--days N` flag to specify how many days ahead to show

**Skill Location:** `~/.claude/skills/cal/`

### `/cal-add` Skill - Add Calendar Events

A Claude Code skill that creates new events on the CalDAV server.

**Usage:**
```bash
# Simple event
/cal-add "Meeting" "2026-06-01 14:00"

# With duration and location
/cal-add "Lunch" "2026-06-01 12:00" "2026-06-01 13:00" "Cafe"

# All-day event
/cal-add "Vacation" "2026-07-01" "2026-07-07" "Paris"
```

**Configuration:**
- Credentials stored in `~/.claude/skills/cal-add/.env`
- Requires: `CALDAV_URL`, `CALDAV_USER`, `CALDAV_PASS`

**Skill Location:** `~/.claude/skills/cal-add/`

### `/cal-remove` Skill - Remove Calendar Events

A Claude Code skill that removes events from the CalDAV server.

**Usage:**
```bash
# List all events with UIDs
/cal-remove --list

# Remove specific event by UID
/cal-remove --uid "abc123-def456-..."

# Remove by partial summary match
/cal-remove --summary "Meeting"

# Remove by exact summary match
/cal-remove --exact "Team Meeting"
```

**Configuration:**
- Credentials stored in `~/.claude/skills/cal-remove/.env`
- Requires: `CALDAV_URL`, `CALDAV_USER`, `CALDAV_PASS`

**Skill Location:** `~/.claude/skills/cal-remove/`

### `/cal-edit` Skill - Edit Calendar Events

A Claude Code skill that modifies existing events on the CalDAV server.

**Usage:**
```bash
# List all events
/cal-edit --list

# Edit event properties
/cal-edit <uid> --summary "New Title"
/cal-edit <uid> --start "2026-06-01 15:00" --end "2026-06-01 16:00"
/cal-edit <uid> --location "New Location"
```

**Configuration:**
- Credentials stored in `~/.claude/skills/cal-edit/.env`
- Requires: `CALDAV_URL`, `CALDAV_USER`, `CALDAV_PASS`

**Skill Location:** `~/.claude/skills/cal-edit/`

**About Claude Code Skills:**
Skills are custom slash commands for Claude Code (Anthropic's CLI tool). They're defined with SKILL.md files and can invoke Python scripts with specified arguments. Skills make it easy to create reusable commands for common tasks.

## Security

**Authentication:**
- CalDAV username stored in: `/etc/radicale/users`, `~/.claude/skills/cal-add/.env`, `~/.claude/skills/cal-remove/.env`, `~/.claude/skills/cal-edit/.env`, `.bashrc`
- Password: bcrypt hashed in htpasswd format
- Required for all calendar access

**SSL/TLS:**
- Let's Encrypt certificates with auto-renewal
- HSTS enabled (max-age=31536000)
- HTTP to HTTPS redirect enforced

**Network:**
- Radicale binds to localhost (127.0.0.1:5232) only
- nginx reverse proxy provides external access
- Firewall: ports 80 and 443 open
- All CalDAV traffic over HTTPS

**Access Control:**
- Radicale rights: `owner_only` (users can only access their own calendars)
- Web UI disabled
- Directory listings disabled

**Data Storage:**
- Calendar data: `/var/lib/radicale/collections`
- Owned by `radicale:radicale` user/group

## FAQ

**Q: I get "Directory listings not supported" when visiting the URL**
A: Normal. Radicale is designed for CalDAV clients, not web browsers.

**Q: How do I access the calendar in a browser?**
A: Use the calendar URL with authentication. Safari can sometimes open iCal feeds directly.

**Q: Why the UUID in the URL?**
A: Radicale uses UUIDs for calendars. The `/calendar.ics` redirect provides a simpler alias.

**Q: Is there a web UI?**
A: Disabled for security. Use /cal-add and /cal skills instead.

**Q: Can I use this with iCloud/Apple Calendar?**
A: Yes! You can add, edit, and delete events on the CalDAV server using iCloud/Apple Calendar as the client. Note that iCloud requires both CalDAV (calendar) and CardDAV (contacts) when adding accounts, so you may need to provide a CardDAV endpoint even if you only want to use the calendar functionality.

**Q: Is the calendar public?**
A: No, authentication is required for all access.

**Q: How do I change the password?**
A: Update `/etc/radicale/users` with htpasswd, then update `~/.claude/skills/cal-add/.env` and `.bashrc`, and restart Radicale.

**Q: What are Claude Code skills?**
A: Custom slash commands that extend Claude Code's functionality. See the "Claude Code Skills" section above for details on the /cal and /cal-add skills.
