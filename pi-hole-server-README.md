# Pi-hole Home Server

Network-wide DNS ad-blocking server built on repurposed hardware.

## Overview

I repurposed an old HP desktop (Pentium Dual-Core E5500, 4GB RAM) into a dedicated home server running Xubuntu 24.04 LTS, then deployed Pi-hole to provide network-wide DNS filtering and ad-blocking for every device on my home network — no per-device configuration needed.

## What I Built

- **Hardware:** Old HP desktop, repurposed rather than bought new
- **OS:** Xubuntu 24.04 LTS
- **Networking:** Assigned a static IP (`192.168.0.198`) via DHCP reservation on the router (Tenda F3)
- **DNS:** Configured the router's Preferred DNS to point to Pi-hole, with the router itself as fallback Alternative DNS — so if Pi-hole ever goes down, the network doesn't lose internet access
- **Remote access:** Set up SSH/SFTP for file transfer between the server and other devices on the network (phone, laptop)

## Always-On Configuration

Pi-hole is my network's DNS server, so every device loses name resolution whenever the server is off. It therefore needs to stay awake and come back on its own after a power cut.

- **Sleep blocked at the OS level:** masked the sleep, suspend and hibernate targets so nothing can put the server to sleep
  ```bash
  sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
  ```
- **Xfce Power Manager:** system sleep/hibernate set to *Never* (the display is allowed to turn off, since that doesn't affect the server)
- **BIOS:** AC power recovery set to *Power On*, so the server restarts by itself after a power outage
- **Services start on boot:** confirmed with
  ```bash
  systemctl is-enabled ssh tailscaled filebrowser pihole-FTL
  ```

**Verification**
- [ ] Reboot test: after `sudo reboot`, SSH, Pi-hole and File Browser all respond without manual action
- [ ] Idle test: server still reachable after being left idle for an hour or more
- [ ] Power-loss test: server comes back on by itself after the power is cut and restored


## Network Diagram

```
Internet
   │
Tenda F3 Router (192.168.0.1)
   │  - Preferred DNS → Pi-hole
   │  - Alternative DNS → itself (fallback)
   │
Pi-hole Server (192.168.0.198)
   │  - Xubuntu 24.04 LTS
   │  - Dashboard: 192.168.0.198/admin
   │
   ├── Laptop
   ├── Phone
   └── Other LAN devices
```

## Screenshots

**1. Pi-hole Admin Dashboard** — live query stats and blocked domain count
<img width="1359" height="699" alt="Pi-hole admin dashboard" src="https://github.com/user-attachments/assets/af78809e-be31-4dee-a430-bfaca72d3019" />

**2. Router DHCP Reservation** — static IP assignment for the server
<img width="1356" height="244" alt="Router DHCP reservation page" src="https://github.com/user-attachments/assets/57bf9041-d717-469a-a29a-051bb59c51fa" />

**3. Router DNS Settings** — Pi-hole set as primary, router as fallback
<img width="1051" height="405" alt="Router DNS settings page" src="https://github.com/user-attachments/assets/e11c1894-e4d2-4504-a6e9-7012ffee2613" />

**4. System Specs** — confirms it's running on constrained hardware (4GB RAM)
<img width="760" height="390" alt="system info" src="https://github.com/user-attachments/assets/53c6f5d3-f04f-4750-ad6d-92273ce8220d" />

**5. SSH/SFTP Session** — remote access and file transfer to the server
<img width="1365" height="467" alt="sftp_session" src="https://github.com/user-attachments/assets/d52268eb-d270-4820-a7a7-c4d6aa521a5d" />


**6. Pi-hole Query Log** — real blocked requests in action
<img width="1302" height="696" alt="Pi hole query blocks" src="https://github.com/user-attachments/assets/4ce8b396-329d-458c-8bda-9be8c3aa3602" />


      **Upstream DNS servers** 
      <img width="1046" height="536" alt="image" src="https://github.com/user-attachments/assets/86e41cff-1a9b-4c22-83f5-1afa6c2d0f7a" />




## What I Learned

- How DNS resolution works at the network level and why sinkholing at the DNS layer is more efficient than per-device ad-blockers
- DHCP reservations vs static IP configuration — trade-offs of each
- Basic Linux server administration on constrained hardware (4GB RAM)
- Setting up secure remote access (SSH/SFTP) on a home network

  ## Troubleshooting: DNS Outage

**Symptom:** the server itself lost working name resolution. `ping 8.8.8.8` succeeded, but `ping google.com` failed with "Temporary failure in name resolution". That pattern means the internet connection was fine and DNS was the broken layer.

**Diagnosis**
1. Compared pinging an IP address against pinging a name, which isolated the problem to DNS
2. `nslookup google.com 8.8.8.8` worked, but `nslookup google.com 127.0.0.1` (Pi-hole) returned `REFUSED`. Pi-hole was running (`systemctl status pihole-FTL`) but declining to answer
3. `sudo pihole-FTL --config dns.upstreams` showed Pi-hole had only **one** upstream DNS server configured
4. `resolvectl status` and `cat /etc/resolv.conf` showed the server's own resolver file had been rewritten by Tailscale and pointed at Tailscale's resolver (`100.100.100.100`) instead of the normal local resolver

**Root causes**
- Pi-hole depended on a single upstream DNS server, so one failing upstream made it refuse every query
- Tailscale's DNS override on the server changed where the server's own lookups went

**Fixes**
```bash
# Give Pi-hole two upstream resolvers
sudo pihole-FTL --config dns.upstreams '["1.1.1.1","8.8.8.8"]'
sudo systemctl restart pihole-FTL

# Stop Tailscale overriding this machine's DNS
sudo tailscale set --accept-dns=false

# Restore Ubuntu's default resolver setup
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

**Result:** name resolution works again on the server and across the network, and Tailscale remote access is unaffected.

**Takeaways**
- Test one layer at a time: IP connectivity, then DNS through the system resolver, then DNS against a specific server
- Because Pi-hole is the network's DNS, a single misconfiguration can look like "the internet is down" for every device
- Redundancy matters: two upstream resolvers instead of one, plus the router as a fallback DNS server on the network




 **upstream DNS servers**     
<img width="1046" height="536" alt="image" src="https://github.com/user-attachments/assets/f5ac3485-8f08-478f-83a5-557ebfed4b67" />


## What I Learned

- How DNS resolution works at the network level and why sinkholing at the DNS layer is more efficient than per-device ad-blockers
- DHCP reservations vs static IP configuration — trade-offs of each
- Basic Linux server administration on constrained hardware (4GB RAM)
- Setting up secure remote access (SSH/SFTP) on a home network


## Next Steps

- Add a fallback DNS resolver (e.g. Unbound) for full recursive resolution instead of relying on upstream DNS
- Set up log rotation and monitoring for the Pi-hole instance
- Document the SSH hardening steps (key-based auth, disabling password login)
