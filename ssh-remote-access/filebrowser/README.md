# File Browser — Remote File Access for My Home Server

A self-hosted web file manager running on my Pi-hole home server, reachable from my phone and laptop over a private Tailscale network. No cloud storage and no ports exposed to the internet.

## Overview

I wanted to upload and download files to my home server from anywhere, without going through SSH/SFTP each time and without handing my files to a third-party cloud service. I installed [File Browser](https://github.com/filebrowser/filebrowser), a lightweight web-based file manager, and ran it as a systemd service on my Xubuntu server (old HP desktop, 4GB RAM). It's a single binary, so it suits the limited hardware, and it doesn't need Docker.

## How it's reached

```
Phone / Laptop (Tailscale on)
        │   encrypted WireGuard tunnel
        ▼
Home server  ──►  File Browser (port 8080)  ──►  ~/shared only
```

Two layers of protection:
1. **Network layer:** the service is only reachable from my home LAN and devices enrolled in my Tailscale network. There is no port forwarding on the router, so it isn't reachable from the public internet.
2. **Application layer:** a login is required, and sign-up is disabled.

## What I built

- Installed File Browser using the official install script
- Created a dedicated `~/shared` folder as the only location File Browser can see
- Initialised its database and configuration (address, port, root folder)
- Created an admin account with a strong password (File Browser enforces a minimum length of 12)
- Wrote a systemd unit so it starts on boot and restarts on failure, running as my normal user rather than root

```ini
[Unit]
Description=File Browser
After=network.target

[Service]
User=<username>
ExecStart=/usr/local/bin/filebrowser -d /var/lib/filebrowser/filebrowser.db
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Security decisions

| Setting | Value | Why |
|---|---|---|
| Root folder | `~/shared` | Keeps `~/.ssh` and the rest of my home directory out of reach of the web interface |
| Sign up | Disabled | Nobody can create their own account on the login page |
| Exec Enabled | `false` | The web interface can't run shell commands on the server |
| Follow External Symlinks | `false` | A link inside `shared` can't be used to reach files outside it |
| Runs as | Normal user, not root | A compromise of the service wouldn't give root access |
| Exposure | LAN + Tailscale only | No router port forwarding, so no internet exposure |

The interface is plain HTTP. Over Tailscale that traffic travels inside an encrypted tunnel, but on my home Wi-Fi it is not encrypted. That's an accepted trade-off for a home lab, and I note it as a next step below.

## Testing

| Test | Result |
|---|---|
| Open the interface from the home network | ✅ Loads, login works |
| Open it from my phone on mobile data with Tailscale on | ✅ Loads, login works |
| Upload a photo from the phone | ✅ File appeared in `~/shared` on the server |
| Download the photo back to the phone | ✅ Downloaded correctly |



## Troubleshooting notes

**Login failed ("wrong credentials").** The service was running, but I hadn't created an account with my own credentials. File Browser locks its database while running, so I stopped the service, listed users with `users ls`, and set the username and password with `users update`, then restarted it.

**Tailscale "Out of sync" message.** Tailscale showed this when my client couldn't reach its coordination server. Restarting the connection (`tailscale down` then `tailscale up` on the server, and reconnecting the phone) cleared it. If it recurs, I'll check the Pi-hole query log for a blocked Tailscale domain.

## What I learned

- How a self-hosted web app is run as a proper service with systemd (start on boot, restart on failure)
- Why the root folder matters: scoping an app to one directory limits the damage if it's ever compromised
- Defense in depth: a private network layer plus an application login, instead of relying on one
- Reading a config dump (`config cat`) to audit security settings rather than assuming defaults
- Diagnosing a service problem by working through logs, status and configuration step by step

## Next steps

- Add HTTPS so traffic is encrypted on the home LAN too (for example via Tailscale's built-in HTTPS support)
- Back up `~/shared` to a second drive or device
- Add `fail2ban` and restrict port 8080 with `ufw` to the LAN and Tailscale ranges
- Try Syncthing for automatic phone-to-server photo sync
