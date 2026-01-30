# FileBrowser File Manager

Self-hosted file management and sharing platform for Ursine project at `https://files.ber.computer`.

## Overview

- **Service**: FileBrowser v2.56.0
- **URL**: https://files.ber.computer
- **Purpose**: File sharing and management for Claude Code skills
- **Tech**: Go-based single binary

## Features

- **Web File Manager**: Beautiful web UI for file management
- **Authentication**: User-based access control
- **File Operations**: Upload, download, rename, delete, move, copy
- **Sharing**: Generate shareable links with expiration
- **Multiple Users**: Fine-grained permissions per user
- **API Access**: RESTful API for programmatic access
- **Lightweight**: Single binary, low resource usage

## Quick Start

### Access

Visit: `https://files.ber.computer`

**Credentials:**
- Stored in `/root/self-hosted/filebrowser/.env`
- Available as environment variables: `$FILEBROWSER_USERNAME`, `$FILEBROWSER_PASSWORD`
- Also exported in `/root/.bashrc`

⚠️ **IMPORTANT**: Credentials are stored in `.env` file - keep it secure!

### Upload Files via Web UI

1. Login to FileBrowser
2. Navigate to desired directory
3. Click "Upload" button or drag & drop files
4. Right-click file → "Share" to get shareable link

### Upload Files via API

```bash
# Upload a file
curl -X POST https://files.ber.computer/api/resources/files/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@/path/to/file.mp3"
```

## Configuration

### Service Management

```bash
# Start service
systemctl start filebrowser

# Stop service
systemctl stop filebrowser

# Restart service
systemctl restart filebrowser

# Check status
systemctl status filebrowser

# View logs
journalctl -u filebrowser -f
```

### Config Location

- **Database**: `/root/self-hosted/filebrowser/filebrowser.db`
- **Files Root**: `/root/share/` (outside git-tracked directory)
- **Binary**: `/usr/local/bin/filebrowser`
- **Service**: `/etc/systemd/system/filebrowser.service`

### Modify Configuration

```bash
cd /root/self-hosted/filebrowser

# Change settings
filebrowser config set --port 8080
filebrowser config set --root /path/to/files

# View all settings
filebrowser config cat

# Restart after changes
systemctl restart filebrowser
```

## User Management

### Add User

```bash
filebrowser users add <username> <password> --perm.admin
```

### List Users

```bash
filebrowser users ls
```

### Update User

```bash
filebrowser users update <username> --password <new_password>
```

### Delete User

```bash
filebrowser users rm <username>
```

## API Usage

### Get Authentication Token

```bash
# Login to get token (using environment variables)
curl -X POST https://files.ber.computer/api/login \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$FILEBROWSER_USERNAME\",\"password\":\"$FILEBROWSER_PASSWORD\"}"
```

Response:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Upload File

```bash
curl -X POST https://files.ber.computer/api/resources/files/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@audio.mp3"
```

### List Files

```bash
curl https://files.ber.computer/api/resources/ \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Share File

```bash
curl -X POST https://files.ber.computer/api/resources/share/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "/audio.mp3",
    "expires": "2026-12-31T23:59:59Z",
    "password": ""
  }'
```

## Claude Code Integration

### Upload Script

The upload script at `~/.claude/skills/file-upload/upload.sh` should use environment variables:

```bash
#!/bin/bash
set -e

# Load credentials from environment
source /root/self-hosted/filebrowser/.env

# Get token
TOKEN=$(curl -s -X POST "$FILEBROWSER_URL/api/login" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$FILEBROWSER_USERNAME\",\"password\":\"$FILEBROWSER_PASSWORD\"}" \
  | jq -r '.token')

# Upload file
curl -X POST "$FILEBROWSER_UPLOAD_URL" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@$1" \
  | jq -r '.url // "https://files.ber.computer/" + (.path | sub("^/";""))'
```

## Security

### SSL/TLS

Currently using self-signed certificate. To enable Let's Encrypt:

```bash
# Temporarily allow HTTP
# Update nginx to allow HTTP on port 80

# Get certificate
certbot certonly --nginx -d files.ber.computer

# Update nginx to use new certificates
ssl_certificate /etc/letsencrypt/live/files.ber.computer/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/files.ber.computer/privkey.pem;

# Reload nginx
systemctl reload nginx
```

### Password Management

Change admin password:
```bash
filebrowser users update admin --password new_secure_password_here
```

### Permissions

FileBrowser supports fine-grained permissions:
- **Admin**: Full access including user management
- **Execute**: Execute files
- **Create**: Upload/create new files
- **Rename**: Rename files
- **Modify**: Edit file contents
- **Delete**: Remove files
- **Share**: Generate shareable links
- **Download**: Download files

## Troubleshooting

### Service not starting

```bash
# Check status
systemctl status filebrowser

# View logs
journalctl -u filebrowser -n 50

# Check if port is in use
ss -tlnp | grep 8080
```

### Can't access via nginx

```bash
# Check nginx status
systemctl status nginx

# Test nginx config
nginx -t

# Check nginx logs
tail -f /var/log/nginx/files.ber.computer-error.log
```

### Files not appearing

Check the root directory setting:
```bash
filebrowser config cat | grep Root
```

## Maintenance

### Backup Database

```bash
cp /root/self-hosted/filebrowser/filebrowser.db /backup/filebrowser-$(date +%Y%m%d).db
```

### Backup Files

```bash
tar czf /backup/files-browser-$(date +%Y%m%d).tar.gz /root/self-hosted/filebrowser/data/
```

### Update FileBrowser

```bash
# Download latest version
cd /tmp
curl -fsSL https://github.com/filebrowser/filebrowser/releases/latest/download/linux-amd64-filebrowser.tar.gz -o filebrowser.tar.gz
tar xvfz filebrowser.tar.gz

# Stop service
systemctl stop filebrowser

# Replace binary
mv filebrowser /usr/local/bin/filebrowser
chmod +x /usr/local/bin/filebrowser

# Start service
systemctl start filebrowser
```

## Resources

- **Official Docs**: https://filebrowser.org/
- **GitHub**: https://github.com/filebrowser/filebrowser
- **Features**: https://filebrowser.org/features

## Integration with Ursine

FileBrowser is integrated with Ursine's Claude Code skills for:
- Uploading generated audio files
- Sharing images and media
- Temporary file hosting
- Quick file sharing in conversations

Files are organized under `/root/self-hosted/filebrowser/data/` and can be managed via the web UI or API.
