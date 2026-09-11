# Switching Lab: MAC Address Learning Across 3 Switches

## Objective

Build a switched network in Cisco Packet Tracer using three switches, and observe how switches dynamically learn and store MAC addresses as traffic flows through the network.

## Topology

```
PC0 ---Fa0/1        Fa0/1--- PC2       Fa0/1---PC4
        \\          /              \\
        SW1 --Gig0/1==Gig0/1--- SW2 --Gig0/2==Gig0/1--- SW3
        /                              \\             \\
PC1 ---Fa0/2                    PC3---Fa0/2      PC5---Fa0/2
```

- **SW1** — connects PC0 (Fa0/1) and PC1 (Fa0/2), uplinked to SW2 via Gig0/1
- **SW2** — connects PC2 (Fa0/1) and PC3 (Fa0/2), uplinked to SW1 via Gig0/1 and SW3 via Gig0/2
- **SW3** — connects PC4 (Fa0/1) and PC5 (Fa0/2), uplinked to SW2 via Gig0/1

All 6 PCs (192.168.0.11–192.168.0.16) share the same subnet (255.255.255.0), with no VLANs or routing involved — a single flat Layer 2 network.

## What I Did

### 1. Built the topology
Placed 3 switches and 6 PCs (2 per switch), cabled with straight-through Ethernet, and addressed all hosts in the same subnet.

<img width="1350" height="440" alt="image" src="https://github.com/user-attachments/assets/ac77260e-ab42-4804-bca2-748b5bbf9df2" />


### 2. Observed the MAC address table populate
- Confirmed the table starts empty (or near-empty) before any traffic.
- Sent pings between PCs and watched `show mac address-table` fill in on each switch.
- Verified that switches only learn MAC addresses from traffic they actually witness — learning is reactive, not automatic network-wide discovery.

**Example output from SW1:**
```
Vlan    Mac Address       Type       Ports
----    -----------       ----       -----
1       0000.0c51.bb1d    DYNAMIC    Fa0/1
1       0001.970d.9271    DYNAMIC    Fa0/2
1       0000.f974.2c19    DYNAMIC    Gig0/1
```

### 3. Identified direct vs. uplink learning
- MACs of directly connected PCs (PC0, PC1) appeared on their physical access ports (Fa0/1, Fa0/2).
- MACs of PCs beyond the switch appeared on the **uplink port** (Gig0/1) — the switch doesn't know *where* the device physically lives, only which port the traffic arrived from.

### 4. Discovered switch-to-switch traffic in the table
One learned MAC on SW1 didn't belong to a PC at all — it belonged to **SW2 itself**, learned via **CDP (Cisco Discovery Protocol)** advertisements that switches send automatically every 60 seconds. This showed that MAC tables reflect *any* device generating frames, not just end hosts.

## Key Takeaways

- MAC address learning is **traffic-driven**: a switch only knows about a MAC once it sees a frame sourced from it.
- Each switch builds its **own independent table** — there's no automatic sharing of MAC tables between switches.
- A learned MAC can point to either a **directly connected port** or an **uplink port**, depending on where the source device physically sits relative to that switch.
- Infrastructure protocols like **CDP** generate their own traffic, so switch MACs can appear in the table alongside host MACs.
- Aging timers (default 300s) remove stale entries, keeping the table accurate as the network changes.

## Next Steps (Lab 3 — Host Port Move)

- Move a PC's cable to a different port (or different switch).
- Use `clear mac address-table dynamic` to force immediate re-learning instead of waiting for the aging timer.
- Re-ping the moved host and confirm the MAC table updates to reflect the new port.
