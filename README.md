# Self-Hosted Services

This directory contains configuration and documentation for self-hosted services on the Ursine server.

## Purpose

These services are self-hosted infrastructure supporting Ursine's custom Claude Code skills. Rather than relying on third-party SaaS products, Ursine maintains its own services for data privacy, control, and reliability.

Each service integrates with one or more Claude Code skills (`/cal`, `/cal-add`, etc.) to provide functionality like calendar management, notifications, and content delivery.

## Services

### [FileBrowser](./filebrowser/)
- **What**: Web-based file manager and sharing platform
- **URL**: https://files.ber.computer
- **Purpose**: File storage and sharing for Ursine Claude Code skills
- **Auth**: Required (username/password)
- **See**: `filebrowser/README.md` for full documentation

### [Radicale](./radicale/)
- **What**: CalDAV/CardDAV server for calendar and contact sync
- **URL**: https://caldav.ber.computer
- **Purpose**: Calendar storage for Ursine Claude Code skills (`/cal` and `/cal-add`)
- **Auth**: Required (bcrypt hashed passwords)
- **See**: `radicale/README.md` for full documentation

## Directory Structure

```
self-hosted/
├── README.md              # This file
├── filebrowser/           # File manager
│   └── README.md          # FileBrowser documentation
└── radicale/              # CalDAV server
    └── README.md          # Radicale documentation
```

## Backup Notes

- FileBrowser data: `/root/self-hosted/filebrowser/data/`
- FileBrowser database: `/root/self-hosted/filebrowser/filebrowser.db`
- Radicale data: `/var/lib/radicale/collections`
- Radicale config: `/etc/radicale/`
- Consider setting up automated backups for any data directories

## Quick Reference

```bash
# FileBrowser status
systemctl status filebrowser

# FileBrowser logs
journalctl -u filebrowser -f

# Radicale status
systemctl status radicale

# Radicale logs
journalctl -u radicale -f

# nginx status (reverse proxy)
systemctl status nginx
```
