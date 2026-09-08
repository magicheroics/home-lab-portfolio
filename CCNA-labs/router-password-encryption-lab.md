# Cisco Router Lab: Enable Password Encryption (R1–R2)

**Topology:** Two Cisco 1941 routers (R1, R2) connected directly via GigabitEthernet0/0.
**Tool:** Cisco Packet Tracer
**Goal:** Configure basic router connectivity and hostnames, then explore how `enable password` is stored in the running configuration with and without `service password-encryption`.

---

## Topology

```
   R1                          R2
[Gig0/0] ------------------ [Gig0/0]
```

---

## Objectives

1. Connect R1 and R2 via their GigabitEthernet0/0 interfaces
2. Set hostnames to match the network diagram
3. Set the enable password to `cisco` on each router
4. Observe the password in the running config (plaintext)
5. Enable password encryption
6. Observe the password again (encrypted)
7. Disable password encryption
8. Observe the password a final time — does disabling encryption reverse it?

---

## Step 1 — Physically Connect the Routers

Connected R1 Gig0/0 to R2 Gig0/0 with a copper straight-through cable.

**Key difference from switches:** router interfaces are **administratively shut down by default**. Even with a good cable, the link stays down until you explicitly enable the interface. Switch ports, by contrast, are enabled out of the box.

On **both** R1 and R2:

```
enable
configure terminal
interface GigabitEthernet0/0
no shutdown
exit
```

Verify with:

```
show interfaces gigabitEthernet0/0
```

Before `no shutdown`:
```
GigabitEthernet0/0 is administratively down, line protocol is down
```

After `no shutdown` (with both ends up and cable good):
```
GigabitEthernet0/0 is up, line protocol is up
```

---

## Step 2 — Set Hostnames

**R1:**
```
enable
configure terminal
hostname R1
```

**R2:**
```
enable
configure terminal
hostname R2
```

---

## Step 3 — Set the Enable Password

On **both** routers:

```
enable password cisco
```

---

## Step 4 — View the Password (Before Encryption)

```
exit
show running-config
```

**Result:**
```
enable password cisco
```

**Is it encrypted? No.** The password is stored in plaintext in the running config. Anyone with read access to the config (or a saved config file) can see it directly.

---

## Step 5 — Enable Password Encryption

On both routers:

```
configure terminal
service password-encryption
exit
```

This applies **Type 7** encryption to all plaintext passwords currently in the config, and automatically encrypts any new ones added afterward.

---

## Step 6 — View the Password (After Encryption)

```
show running-config
```

**Result (example):**
```
enable password 7 094F471A1A0A
```

**Is it encrypted? Yes** — it now shows as Type 7. Note: Type 7 is a weak, reversible cipher (easily cracked with widely available tools), so it protects against casual shoulder-surfing but not a determined attacker.

---

## Step 7 — Disable Password Encryption

```
configure terminal
no service password-encryption
exit
```

---

## Step 8 — View the Password (After Disabling Encryption)

```
show running-config
```

**Result:**
```
enable password 7 094F471A1A0A
```

**Is it still encrypted? Yes.**

### Key Takeaway
`no service password-encryption` only stops the router from encrypting **future** passwords — it does **not** decrypt passwords that are already encrypted. Once a password has been converted to Type 7, it stays that way in the running config regardless of whether the service is later disabled.

---

## Lessons Learned

- Router interfaces default to `shutdown`; switch interfaces do not — always check with `show interfaces` when a link isn't coming up.
- `enable password` stores credentials in plaintext by default — a real security risk if configs are ever backed up, emailed, or pasted into a ticket.
- `service password-encryption` is a low-effort mitigation but uses a weak (Type 7) cipher — for real deployments, `enable secret` (which uses a proper one-way hash) should be preferred over `enable password`.
- Disabling `service password-encryption` is not the same as decrypting existing passwords — it's a one-way ratchet for anything already encrypted.

## Next Steps
- Repeat this lab using `enable secret` instead of `enable password` and compare encryption behavior.
- Add IP addressing to Gig0/0 on both routers and verify connectivity with `ping`.
