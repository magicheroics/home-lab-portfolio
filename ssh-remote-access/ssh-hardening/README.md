# SSH Hardening & Remote Access — Home Server

Securing remote access to my Pi-hole home server with key-based SSH authentication, and making it reachable from outside my home network.

## Overview

My home server (old HP desktop running Xubuntu 24.04 LTS) was already reachable over SSH/SFTP on the local network using a password. Because I also wanted to reach it from my phone on other networks, password login became a real risk: anything reachable remotely gets hit by automated brute-force attempts. I replaced password authentication with SSH key pairs and disabled password login entirely.

## Why keys instead of passwords

- A password can be guessed or brute-forced; a 256-bit ed25519 private key effectively cannot
- The private key never leaves the device it was generated on — only the public key is stored on the server
- With password login disabled, a device without an authorized key is refused outright, no matter what password it tries

## What I did

1. **Generated an ed25519 key pair** on my laptop (`ssh-keygen -t ed25519`)
2. **Copied the public key to the server** with `ssh-copy-id`, which appends it to `~/.ssh/authorized_keys`
3. **Tested key login in a second terminal** while keeping the original session open, so a mistake couldn't lock me out
4. **Generated a separate key pair on my phone** and added its public key to the server's `authorized_keys` — each device gets its own key, so one can be revoked without affecting the other
5. **Hardened the SSH daemon** in `/etc/ssh/sshd_config`:
   ```
   PasswordAuthentication no
   PubkeyAuthentication yes
   PermitRootLogin no
   ```
6. **Checked for override files** in `/etc/ssh/sshd_config.d/` — on Ubuntu 24.04 these can silently override the main config
7. **Restarted SSH** (`sudo systemctl restart ssh`) and **verified the effective setting** with:
   ```
   sudo sshd -T | grep -i passwordauthentication
   ```
   which returned `passwordauthentication no`
8. **Fixed permissions** where needed (`chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys`) — SSH silently ignores `authorized_keys` if it's too permissive

## Remote access from outside the home network

I can now reach the server from my phone while on a different network, authenticating with the phone's key rather than a password. *(Edit this line to name the method you used — e.g. Tailscale VPN — and how it's set up.)*



## What I learned

- How public/private key authentication works, and why the private key never needs to leave the client
- Why I kept an existing session open while testing, and made changes in the right order, to avoid locking myself out of a remote server
- Why `sshd -T` is a better check than reading the config file: it shows the *effective* settings after all overrides
- How `#` comments work in config files: a commented line is ignored, so the default applies
- Each device should have its own key, so access can be revoked per device

## Next steps

- Review `authorized_keys` periodically and remove keys for devices I no longer use
- Add `fail2ban` to block repeated failed connection attempts
- Consider changing the default SSH port or restricting SSH to specific source addresses
- Set up automatic security updates on the server
