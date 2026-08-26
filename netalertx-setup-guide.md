# Home Network Monitoring with NetAlertX + Pi-hole

A writeup of how I added new-device detection and email alerting to my home lab, running on an old HP 500B Microtower (Xubuntu) that already hosts Pi-hole.

## Goal

Get an email notification whenever a new (or previously unknown) device connects to my local network — something my budget router (Tenda F3) doesn't support natively, and something Pi-hole alone can't do since it only sees DNS traffic, not raw network joins.

## Stack

- **Host**: HP 500B Microtower, Xubuntu, already running Pi-hole for DNS/ad-blocking
- **Tool**: [NetAlertX](https://github.com/jokob-sk/NetAlertX) (formerly Pi.Alert) — open-source network scanner/inventory tool
- **Deployment**: Docker, managed via Portainer
- **Notifications**: Email via Gmail SMTP

## Why NetAlertX

Pi-hole only logs DNS queries, so a device could join the network and never even hit Pi-hole if it doesn't use it for DNS. NetAlertX works at the network layer — it ARP-scans the subnet directly, so it catches anything that joins regardless of DNS behavior. It also pairs well with Pi-hole (can pull DHCP/DNS data from it for better device naming).

## Setup

### 1. Docker

Docker was already installed via the official Docker CE repo (not Ubuntu's `docker.io` package — these two conflict if mixed, more on that below).

```bash
docker --version
docker compose version
```

Added my user to the `docker` group to avoid needing `sudo` for every command:

```bash
sudo usermod -aG docker <user>
```

### 2. docker-compose.yml

```yaml
services:
  netalertx:
    image: ghcr.io/jokob-sk/netalertx:latest
    container_name: netalertx
    network_mode: host
    restart: unless-stopped
    cap_add:
      - NET_RAW
      - NET_ADMIN
      - NET_BIND_SERVICE
    volumes:
      - ./data:/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - PORT=20211
      - TZ=Africa/Nairobi
```

Key points:
- `network_mode: host` — required so the container can see real LAN traffic for ARP scanning (default Docker networking isolates it from the host's network).
- `cap_add: NET_RAW, NET_ADMIN, NET_BIND_SERVICE` — required as of recent NetAlertX releases; without these the container fails to start (see Issue #2 below).
- Dashboard is available at `http://<server-ip>:20211`.

```bash
mkdir -p ~/netalertx/data
cd ~/netalertx
docker compose up -d
docker compose logs -f   # watch startup
```

### 3. Email notifications (Settings → Publishers → SMTP)

| Field | Value |
|---|---|
| SMTP server URL | `smtp.gmail.com` |
| SMTP server PORT | `587` |
| SMTP user | Gmail address |
| SMTP password | Gmail **App Password** (not the account password — requires 2-Step Verification enabled) |
| Send email to | destination inbox |
| From email | `NetAlertX <sender@gmail.com>` |
| When to run | `on_notification` (fires automatically whenever there's something to report) |

App Passwords are generated at: https://myaccount.google.com/apppasswords

### 4. What triggers the report

NetAlertX has a built-in notification cycle (not the "Workflows" section — that's only for automation on device records like renaming/deleting). Each scan cycle checks:

```
[Notification] Check if something to report
[Notification] Included sections: ['new_devices', 'down_devices', 'events']
```

If a new device is found, it publishes via whatever channels are enabled under Settings → Publishers (SMTP in this case).

## Issues hit along the way

### 1. `docker.io` / `containerd.io` package conflict
Tried to install Docker via `apt install docker.io docker-compose-v2`, but Docker CE (from Docker's official repo) was already installed. `docker.io` (Ubuntu's own package) conflicts with `containerd.io` (Docker's official package). Fix: just use the existing Docker CE install, don't mix package sources.

### 2. `python3: Operation not permitted` (exit code 126)
Container stuck in a restart loop with the error appearing at container startup, in the `mounts.py` entrypoint script. Root cause: a NetAlertX release update requires explicit container capabilities (`NET_RAW`, `NET_ADMIN`, `NET_BIND_SERVICE`) that aren't part of Docker's default capability set. Fix: add `cap_add` to the compose file (see above). Confirmed via NetAlertX's own release notes — this is a known breaking change.

### 3. Corrupted SMTP password causing `UnicodeDecodeError`
After configuring SMTP, the container crashed while trying to *read back* the stored password (not even reaching Gmail):

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xa5 in position 0: invalid start byte
```

This happened inside NetAlertX's settings-value decoding logic, before the actual SMTP login attempt. Likely caused by a copy-paste artifact when entering the App Password. Fix: cleared the SMTP password field, saved with it empty, then re-entered a freshly generated App Password by typing it manually instead of pasting.

## Result

Confirmed working end to end — a new device joining the network now triggers an email like this:

```
New devices
devName: (unknown)
eveMac: 80:2b:f9:9b:d8:dd
devVendor: Hon Hai Precision Ind. Co.,Ltd.
eveIp: 192.168.0.110
eveDateTime: 2026-08-26T12:16:16+02:00
eveEventType: New Device
```

## NetAlertX basics reference

- **Devices** — the main inventory: MAC, IP, hostname, vendor (from MAC prefix), status (online/offline/new/archived). Name every recognized device here.
- **Monitoring** — scan activity/history/events timeline.
- **Network** — topology view of how devices relate to each other, where available.
- **Settings → Device scanners** — detection methods: ARP scan (primary), Pi-hole integration, naming plugins (AVAHISCAN, NBTSCAN, NSLOOKUP, DIGSCAN).
- **Settings → Publishers** — output channels: email/SMTP, Discord, Telegram, ntfy, MQTT, webhooks.
- **Workflows** — automation rules on device records (update field / delete device) triggered by device events — **not** for notifications.

## Next steps / ideas

- Link NetAlertX to Pi-hole for better device names via DHCP lease data
- Periodically review devices marked "down" for anything unexpected
- Treat this as a living inventory — most useful once the device list is fully populated over a few weeks
