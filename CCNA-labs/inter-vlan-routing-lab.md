# Inter-VLAN Routing Lab — 3 Departments, 4 Switches, Router-on-a-Stick

A hands-on Cisco Packet Tracer lab demonstrating VLAN segmentation and inter-VLAN routing across a small hierarchical network: 3 department VLANs, 4 switches (1 core + 3 access), a router-on-a-stick, and 9 PCs.

## Topology overview

```
                Router (2911)
                      |
                 GigabitEthernet0/0
                      |
               Core Switch (2960)
              /        |        \
        FastEthernet trunks to each access switch
            /          |          \
        SW1           SW2          SW3
      (3 PCs)        (3 PCs)      (3 PCs)
```

- **Router**: Cisco 2911 — single GigabitEthernet0/0 physical link to the core switch, split into 3 subinterfaces (router-on-a-stick).
- **Core switch**: Cisco 2960 — 4 trunk ports (1 to the router, 3 to the access switches). No PCs connect here directly.
- **Access switches (SW1–SW3)**: Cisco 2960 — each has 3 access ports (one PC per VLAN) and 1 trunk uplink to the core switch.
- **PCs**: 9 total, 3 per VLAN, spread one-per-VLAN across each access switch.

## VLAN / department plan

| Department | VLAN ID | Subnet | Gateway (router subinterface) |
|---|---|---|---|
| Sales | 10 | 192.168.10.0/24 | 192.168.10.1 |
| HR | 20 | 192.168.20.0/24 | 192.168.20.1 |
| IT | 30 | 192.168.30.0/24 | 192.168.30.1 |

## IP addressing table

| PC | VLAN | Switch | IP address | Subnet mask | Default gateway |
|---|---|---|---|---|---|
| PC0 | 10 (Sales) | SW1 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC3 | 10 (Sales) | SW2 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 |
| PC6 | 10 (Sales) | SW3 | 192.168.10.12 | 255.255.255.0 | 192.168.10.1 |
| PC1 | 20 (HR) | SW1 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| PC4 | 20 (HR) | SW2 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 |
| PC7 | 20 (HR) | SW3 | 192.168.20.12 | 255.255.255.0 | 192.168.20.1 |
| PC2 | 30 (IT) | SW1 | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| PC5 | 30 (IT) | SW2 | 192.168.30.11 | 255.255.255.0 | 192.168.30.1 |
| PC8 | 30 (IT) | SW3 | 192.168.30.12 | 255.255.255.0 | 192.168.30.1 |

## Why this design

Each access switch has one PC from every department, so every switch — and every uplink — must carry all three VLANs. That's what makes trunking (802.1Q) necessary: a single physical cable can carry tagged traffic for multiple VLANs, so the switch-to-switch and switch-to-router links don't need one cable per VLAN.

Traffic within the same VLAN (e.g. PC0 to PC3, both Sales) is switched — it never needs to leave Layer 2, even though it crosses through the core switch. Traffic between VLANs (e.g. PC0 to PC1, Sales to HR) has to go up to the router, since VLANs are separate broadcast domains by design and only a Layer 3 device can move traffic between them.

## Configuration

### 1. Access switches (SW1, SW2, SW3) — repeat on each

```
enable
configure terminal

vlan 10
 name Sales
vlan 20
 name HR
vlan 30
 name IT
exit

interface fa0/1
 switchport mode access
 switchport access vlan 10

interface fa0/2
 switchport mode access
 switchport access vlan 20

interface fa0/3
 switchport mode access
 switchport access vlan 30

interface fa0/4
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

### 2. Core switch

The core switch has no PCs plugged in directly, but it still needs the VLANs defined in its own VLAN database — otherwise trunked frames tagged for those VLANs get dropped, even if the trunk ports themselves are configured correctly. This was the exact issue that caused ping timeouts during testing (see Troubleshooting below).

```
enable
configure terminal

vlan 10
 name Sales
vlan 20
 name HR
vlan 30
 name IT
exit

interface range fa0/1-3
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

interface gi0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

(`fa0/1-3` = links to the 3 access switches; `gi0/1` = link to the router. The router's 2911 only has Gigabit interfaces, so this link is Gig-to-Gig by hardware necessity, while the switch-to-switch trunks stay FastEthernet.)

### 3. Router — router-on-a-stick

```
enable
configure terminal

interface gi0/0
 no shutdown

interface gi0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface gi0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

interface gi0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
```

`encapsulation dot1Q <vlan>` tells each subinterface which VLAN's tagged frames to accept off the shared trunk link, and strips/adds the 802.1Q tag as traffic passes through. Each subinterface's IP address becomes the default gateway for that VLAN's PCs.

## Verification commands

```
show vlan brief              " on each switch — confirms port-to-VLAN assignment
show interfaces trunk        " on each switch — confirms trunk status + active VLANs
show ip interface brief      " on the router — confirms all subinterfaces are up/up
```

## Troubleshooting notes from this build

**Symptom:** Pings between PCs (both same-VLAN and cross-VLAN) returned "Request timed out."

**Root cause:** The core switch's access-port-to-VLAN mappings and trunk configuration were correct, but VLANs 10, 20, and 30 had never been created in the core switch's own VLAN database. `show interfaces trunk` on the core switch showed the giveaway: "Vlans allowed on trunk" listed `10,20,30`, but "Vlans allowed and active in management domain" showed `none` for every port — meaning the trunk was configured to permit those VLANs, but had nothing valid to actually forward since they didn't exist locally.

**Fix:** Running `vlan 10 / name Sales` (and 20/30) on the core switch resolved it immediately. Every switch that a VLAN's traffic passes through needs that VLAN registered in its database — including switches with no access ports assigned to it, as long as it's carrying trunk traffic for that VLAN.

**Lesson:** A switch with no PCs in VLAN 10 can still forward VLAN 10 traffic over a trunk, but only if VLAN 10 exists in its own VLAN database first. Trunk "allowed" lists control what's permitted; the local VLAN database controls what's forwardable.

## Testing checklist


- [ ] Same-VLAN, same-switch ping (e.g. two PCs on SW1, different VLANs — skip, use same VLAN)
- [ ] Same-VLAN, different-switch ping (e.g. PC0 → PC3, both VLAN 10) — should switch through the core, no router involved
- [ ] Cross-VLAN ping (e.g. PC0 → PC1, VLAN 10 → VLAN 20) — should route through the router's subinterfaces
- [ ] `show vlan brief` on all switches — all 3 VLANs active everywhere
- [ ] `show interfaces trunk` on all switches — all VLANs active and forwarding, not just allowed
- [ ] `show ip interface brief` on router — all subinterfaces up/up




<img width="924" height="518" alt="ksnip_20260902-130613" src="https://github.com/user-attachments/assets/1042a4db-7690-488a-9033-a7ff76eebf58" />



