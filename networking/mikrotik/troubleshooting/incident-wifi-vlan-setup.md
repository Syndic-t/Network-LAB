# Incident Report: WiFi & VLAN Setup Issues

**Date:** June 25, 2026
**Device:** MikroTik RB951Ui-2HnD

---

## Issue 1 — wlan1 Locked by CAPsMAN

**Error:** "managed by CAPsMAN" on every wlan1 configuration attempt

**Cause:** Default config enables CAPsMAN and assigns wlan1 to it. Disabling CAPsMAN via Winbox UI does not release the interface.

**Fix (terminal only):**
```routeros
/interface wireless cap set enabled=no interfaces=""
/caps-man manager set enabled=no
/caps-man provisioning remove [find]
/caps-man manager interface set [find default=yes] forbid=no
```

After this, the red CAPsMAN banner disappears and wlan1 can be configured as a standalone AP.

---

## Issue 2 — Bridge VLAN Entries Created as Dynamic (Read-Only)

**Cause:** VLANs created via the Winbox Bridge → VLANs tab are created as dynamic entries (D flag) and cannot be edited.

**Fix:** Always create bridge VLAN entries via terminal:
```routeros
/interface bridge vlan add bridge=bridgeLocal vlan-ids=10 tagged=bridgeLocal untagged=ether2
/interface bridge vlan add bridge=bridgeLocal vlan-ids=20 tagged=bridgeLocal,ether2 untagged=ether3
```

---

## Issue 3 — VLAN Filtering Dropped All Connectivity

**Cause:** VLAN filtering was enabled before the VLAN table was fully populated. The bridge dropped all untagged traffic.

**Recovery:**
```routeros
/interface bridge set bridgeLocal vlan-filtering=no
```

**Rule:** Always complete the full VLAN table (entries, port PVIDs, frame-types) before enabling VLAN filtering.

---

## Issue 4 — VLAN Interfaces Had Wrong IPs (DHCP-Assigned Instead of Gateway)

**Symptom:** `/ip address print` showed `192.168.10.109/24` and `192.168.20.69/24` on LUMINARA and ONIRICA — DHCP-assigned addresses instead of gateway `.1` addresses.

**Cause:** VLAN interfaces accidentally had DHCP clients configured on them, or IPs were never manually set and the interface picked up a lease.

**Fix:**
```routeros
/ip address remove [find interface=LUMINARA]
/ip address remove [find interface=ONIRICA]
/ip address add address=192.168.10.1/24 interface=LUMINARA
/ip address add address=192.168.20.1/24 interface=ONIRICA
```

**Rule:** VLAN interfaces never self-assign gateway IPs. Always add them manually.

---

## Issue 5 — Hyper-V Refused Second External Virtual Switch

**Error:** `External Ethernet adapter Realtek PCIe GBE Family Controller is already bound to the Microsoft Virtual Switch protocol.`

**Cause:** Only one Hyper-V external virtual switch can be bound to a single physical NIC at a time.

**Fix:**
1. Hyper-V Manager → Virtual Switch Manager
2. Remove existing external switch (Network Sync)
3. Create new external switch `HomeTrunk-MKR` on the same Realtek adapter
4. Assign Hyper-V VMs to `HomeTrunk-MKR` with VLAN ID 20

**Note:** Removing the old switch will temporarily break any VMs using it.

---

## Session 2 Key Rules

1. Use terminal for bridge VLAN entries — Winbox creates read-only dynamic entries
2. Populate the full VLAN table before enabling VLAN filtering
3. VLAN interface gateway IPs must be assigned manually
4. CAPsMAN requires terminal commands to fully release wlan1
5. Only one Hyper-V external switch per physical NIC
