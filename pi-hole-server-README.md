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






## What I Learned

- How DNS resolution works at the network level and why sinkholing at the DNS layer is more efficient than per-device ad-blockers
- DHCP reservations vs static IP configuration — trade-offs of each
- Basic Linux server administration on constrained hardware (4GB RAM)
- Setting up secure remote access (SSH/SFTP) on a home network

## Next Steps

- Add a fallback DNS resolver (e.g. Unbound) for full recursive resolution instead of relying on upstream DNS
- Set up log rotation and monitoring for the Pi-hole instance
- Document the SSH hardening steps (key-based auth, disabling password login)
