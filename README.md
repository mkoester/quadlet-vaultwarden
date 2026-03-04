# quadlet-vaultwarden

Quadlet setup for [Vaultwarden](https://github.com/dani-garcia/vaultwarden) — a lightweight Bitwarden-compatible server (`docker.io/vaultwarden/server`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `vaultwarden.container` | Quadlet unit file |
| `vaultwarden.env` | Default environment variables |
| `vaultwarden.override.env.template` | Template for local overrides (domain, admin token, SMTP) |
| `vaultwarden-backup.service` | Systemd service: SQLite snapshot + rsync of remaining data |
| `vaultwarden-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/vaultwarden -s /usr/sbin/nologin vaultwarden

REPO_URL=https://github.com/mkoester/quadlet-vaultwarden.git
REPO=~vaultwarden/quadlet-vaultwarden
```

```sh
# 2. Enable linger
sudo loginctl enable-linger vaultwarden

# 3. Clone this repo into the service user's home
sudo -u vaultwarden git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u vaultwarden mkdir -p ~vaultwarden/.config/containers/systemd
sudo -u vaultwarden mkdir -p ~vaultwarden/data

# 5. Create .override.env from template and fill in required values
sudo -u vaultwarden cp $REPO/vaultwarden.override.env.template $REPO/vaultwarden.override.env
sudo -u vaultwarden nano $REPO/vaultwarden.override.env

# 6. Symlink all quadlet files from the repo
sudo -u vaultwarden ln -s $REPO/vaultwarden.container ~vaultwarden/.config/containers/systemd/vaultwarden.container
sudo -u vaultwarden ln -s $REPO/vaultwarden.env ~vaultwarden/.config/containers/systemd/vaultwarden.env
sudo -u vaultwarden ln -s $REPO/vaultwarden.override.env ~vaultwarden/.config/containers/systemd/vaultwarden.override.env

# 7. Reload and start
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user daemon-reload
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user start vaultwarden

# 8. Verify
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user status vaultwarden
```

## Configuration

### Environment variables

`vaultwarden.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |
| `ROCKET_PORT` | `8080` | HTTP port inside the container |
| `SIGNUPS_ALLOWED` | `false` | Disable public registration |

`vaultwarden.override.env` (created from template) must set:

| Variable | Description |
|---|---|
| `DOMAIN` | Full HTTPS URL where Vaultwarden is accessible (e.g. `https://vaultwarden.example.com`) |
| `ADMIN_TOKEN` | Token for the admin panel (`/admin`) — leave empty to disable the admin panel |

To apply changes after editing the override file:

```sh
sudo -u vaultwarden nano ~vaultwarden/quadlet-vaultwarden/vaultwarden.override.env
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user restart vaultwarden
```

## Reverse proxy (Caddy)

Vaultwarden requires HTTPS to function (browsers restrict the Web Crypto API to secure contexts). Add a site block to your Caddyfile:

```
vaultwarden.example.com {
    reverse_proxy localhost:8080
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Vaultwarden stores all data in `/data` inside the container (`~vaultwarden/data/` on the host): the SQLite vault database, attachments, sends, and the icon cache. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

The backup service uses `sqlite3 .backup` for a consistent online snapshot of the vault database, and `rsync` for the remaining files (attachments, sends, icon cache).

```sh
# 1. Install sqlite3 on the host if not already present
which sqlite3 || sudo apt-get install -y sqlite3

# 2. Create backup staging directories (owned by vaultwarden, readable by backup-readers group)
sudo mkdir -p /var/backups/vaultwarden/data
sudo chown -R vaultwarden:backup-readers /var/backups/vaultwarden
sudo chmod -R 750 /var/backups/vaultwarden

# 3. Symlink the backup service and timer from the repo
sudo -u vaultwarden mkdir -p ~vaultwarden/.config/systemd/user
sudo -u vaultwarden ln -s $REPO/vaultwarden-backup.service ~vaultwarden/.config/systemd/user/vaultwarden-backup.service
sudo -u vaultwarden ln -s $REPO/vaultwarden-backup.timer ~vaultwarden/.config/systemd/user/vaultwarden-backup.timer

# 4. Enable and start the timer
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user daemon-reload
sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user enable --now vaultwarden-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@vaultwarden-host:/var/backups/vaultwarden/ /path/to/local/backup/vaultwarden/
```

This pulls both the vault database snapshot (`db.sqlite3`) and the remaining data (`data/` — attachments, sends, icon cache).

## Notes

- Port `8080` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- All persistent data is stored at `~vaultwarden/data/` on the host.
- `DOMAIN` must be set to the HTTPS URL before first start.
- `ADMIN_TOKEN` should be an Argon2 hash for security. Generate one with:
  ```sh
  echo -n "yourpassword" | argon2 "$(openssl rand -base64 32)" -id -k 65540 -t 3 -p 4 | sed 's#\$#$$#g'
  ```
  The admin panel is accessible at `/admin`. Leave `ADMIN_TOKEN` empty to disable it entirely.
- `SIGNUPS_ALLOWED=false` disables public registration by default. Invite users from the admin panel instead.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u vaultwarden XDG_RUNTIME_DIR=/run/user/$(id -u vaultwarden) systemctl --user enable --now podman-image-prune@30.timer
  ```
