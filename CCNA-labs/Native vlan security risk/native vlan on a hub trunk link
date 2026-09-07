# Native VLAN Security Risk — Hub on a Trunk Link

A Cisco Packet Tracer lab demonstrating how a hub placed on a trunk link leaks native VLAN traffic to any device connected to it — even devices with no VLAN assignment and no relationship to the network's switches at all.

## Motivation

Small organizations that can't afford enough managed switches sometimes extend their network using a cheap hub instead. This lab tests whether that's actually safe by deliberately introducing a hub into a trunk link between two switches and seeing what an unmanaged device plugged into that hub can reach.

## Topology

```
        SW0 (2950-24)                                    SW1 (2950-24)
   ┌──────┴──────┐                                   ┌──────┴──────┐
   │             │                                   │             │
 Fa0/1         Fa0/2                                Fa0/1         Fa0/2
   │             │                                   │             │
  PC0           PC1                                 PC2           PC3
 VLAN 10      VLAN 20                              VLAN 10      VLAN 20
192.168.10.1 192.168.20.1                        192.168.10.2 192.168.20.2

        Fa0/24 (trunk) ──── Hub0 ──── (trunk) Fa0/24
                               │
                              Fa1
                               │
                         Attacker PC (PC4)
                          192.168.10.99
                        (no switch, no VLAN)
```

- **SW0 / SW1:** Cisco 2950-24, each with two access ports (VLAN 10 and VLAN 20) and one trunk port
- **Hub0:** a plain Layer 1 hub sitting between the two switches' trunk ports — carrying all trunk traffic between them
- **Attacker PC (PC4):** plugged directly into a free hub port, with no switch and no VLAN assignment of its own

## VLAN plan

| VLAN | Name | Subnet |
|---|---|---|
| 10 | Trusted | 192.168.10.0/24 |
| 20 | Guest | 192.168.20.0/24 |

## IP addressing

| Device | VLAN context | IP address | Subnet mask |
|---|---|---|---|
| PC0 | VLAN 10 (SW0) | 192.168.10.1 | 255.255.255.0 |
| PC1 | VLAN 20 (SW0) | 192.168.20.1 | 255.255.255.0 |
| PC2 | VLAN 10 (SW1) | 192.168.10.2 | 255.255.255.0 |
| PC3 | VLAN 20 (SW1) | 192.168.20.2 | 255.255.255.0 |
| PC4 (Attacker) | On hub — no VLAN | 192.168.10.99 | 255.255.255.0 |

The Attacker's IP was deliberately set to match the VLAN 10 subnet — that choice is what makes the test meaningful. See "Why the native VLAN matters" below.

## Configuration

Applied identically on both SW0 and SW1:

```
vlan 10
 name Trusted
vlan 20
 name Guest
exit

interface fa0/1
 switchport mode access
 switchport access vlan 10

interface fa0/2
 switchport mode access
 switchport access vlan 20

interface fa0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 10
```

## Why the native VLAN matters

Every 802.1Q trunk has a native VLAN — traffic belonging to it crosses the trunk **untagged**, while every other VLAN's traffic is tagged. This isn't something added for the lab; it's default trunk behavior (VLAN 1 unless changed, set to VLAN 10 here).

That distinction is the entire vulnerability this lab demonstrates:

- The Attacker's NIC is a completely ordinary network card — it can only interpret standard, untagged Ethernet frames. It has no concept of an 802.1Q tag.
- **VLAN 10 (native) traffic** crosses the trunk untagged, so to any device it looks like normal Ethernet — including the Attacker's NIC. It reads it fine.
- **VLAN 20 traffic** crosses the trunk tagged. The Attacker's NIC can't parse the tag, so even though the frame physically reaches it via the hub, it can't use it.

A hub has no VLAN awareness at all — it doesn't enforce anything, it just repeats every bit it receives out every other port. So when a hub sits on a trunk, whatever happens to be native VLAN traffic becomes readable by anything plugged into that hub, VLAN membership or not.

## Test results

| # | Test | Path | Expected | Result |
|---|---|---|---|---|
| 1 | Same VLAN, different switch | PC0 (VLAN 10) → PC2 (VLAN 10) | Success | ✅ Successful |
| 2 | Different VLAN, same switch | PC0 (VLAN 10) → PC1 (VLAN 20) | Fail | ❌ Failed |
| 3 | Different VLAN, different switch | PC0 (VLAN 10) → PC3 (VLAN 20) | Fail | ❌ Failed |
| 4 | **Attacker reaches native VLAN** | PC4 (no VLAN, on hub) → PC0 (VLAN 10) | Success (the vulnerability) | ✅ Successful |

Tests 1–3 confirm the switches enforce VLAN isolation correctly, both locally and across the trunk. Test 4 is the finding: a device with no VLAN assignment, connected only to a hub, successfully reached a "trusted" VLAN 10 host — something that should not be possible in a properly segmented network.

Packet Tracer's Simulation mode confirmed the packet's actual path: **PC4 → Hub0 → Switch0 → PC0**, with a "Successful" ICMP result logged in the PDU list — ruling out any ambiguity about whether the ping genuinely completed.

## The finding

Placing a hub on a trunk link defeats VLAN segmentation for the native VLAN specifically. Any device plugged into that hub — regardless of whether it belongs to the network, has any VLAN configuration, or sits behind a switch at all — can read and respond to native VLAN traffic, because that traffic is never tagged in the first place. This is not a hub-specific flaw in general (tagged VLANs, like VLAN 20 in this lab, are unaffected) — it's specifically a native-VLAN-on-untrusted-media risk.

## Mitigation

- **Never assign the native VLAN to a VLAN carrying real traffic.** Best practice is to set the native VLAN to an unused, dummy VLAN (e.g. VLAN 999) so nothing meaningful ever rides the trunk untagged.
- **Avoid hubs on trunk links entirely** where possible — a switch enforces VLAN boundaries; a hub cannot.
- If a hub must be used for cost reasons, keep it strictly within a single VLAN's access-layer wiring, never on a trunk carrying multiple VLANs.

## What I learned

- The difference between VLAN enforcement (switches, by design) and VLAN "isolation" as a side effect of tagging (hubs, by accident)
- Why the native VLAN is a meaningful security consideration, not just a trunk configuration default
- How to use Packet Tracer's Simulation mode and PDU list to get definitive pass/fail evidence, not just visual inspection
- Real-world justification for why native VLAN mismatch/misuse is a commonly cited hardening checklist item in actual network audits
