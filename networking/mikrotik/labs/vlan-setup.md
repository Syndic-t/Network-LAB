# Lab: WiFi Restore + VLAN Setup (Session 2)

**Date:** June 25, 2026
**Device:** MikroTik RB951Ui-2HnD
**Follows:** [rb951-homelab-setup.md](rb951-homelab-setup.md)

## Objectives

1. Restore wlan1 as a standalone access point (released from CAPsMAN)
2. Create VLAN 10 (LUMINARA) for the physical PC — 192.168.10.0/24
3. Create VLAN 20 (ONIRICA) for Hyper-V VMs — 192.168.20.0/24

## Final Network Layout

```
ether1 ──── WAN (192.168.0.105) → upstream
bridgeLocal (192.168.88.1) → management / WiFi
  ├── VLAN 10 / LUMINARA (192.168.10.1) → Physical PC
  └── VLAN 20 / ONIRICA (192.168.20.1)  → Hyper-V VMs
```

### Port Map

| Port | Mode | VLANs | Connected To |
|---|---|---|---|
| ether1 | WAN | — | Unmanaged switch |
| ether2 | Trunk | VLAN 10 untagged + VLAN 20 tagged | PC + Hyper-V |
| ether3 | Access | VLAN 20 untagged | Reserved (dedicated Hyper-V port) |
| wlan1 | AP Bridge | Management | WiFi backup |

## Part 1 — WiFi Setup

### Release wlan1 from CAPsMAN

```routeros
/interface wireless cap set enabled=no interfaces=""
/caps-man manager set enabled=no
/caps-man provisioning remove [find]
/caps-man manager interface set [find default=yes] forbid=no
```

### Configure Standalone AP

```routeros
/interface wireless enable wlan1
/interface wireless set wlan1 mode=ap-bridge band=2ghz-b/g/n frequency=auto ssid=YourSSID
```

Set WPA2 in Winbox: Wireless → Security Profiles → default → set WPA2 PSK and passphrase → assign to wlan1.

## Part 2 — VLAN Setup

### Step 1: Create VLAN interfaces

```routeros
/interface vlan add name=LUMINARA vlan-id=10 interface=bridgeLocal
/interface vlan add name=ONIRICA vlan-id=20 interface=bridgeLocal
```

### Step 2: Assign gateway IPs (must be done manually)

```routeros
/ip address add address=192.168.10.1/24 interface=LUMINARA
/ip address add address=192.168.20.1/24 interface=ONIRICA
```

### Step 3: Create bridge VLAN entries via terminal (NOT Winbox UI)

```routeros
/interface bridge vlan add bridge=bridgeLocal vlan-ids=10 tagged=bridgeLocal untagged=ether2
/interface bridge vlan add bridge=bridgeLocal vlan-ids=20 tagged=bridgeLocal,ether2 untagged=ether3
```

### Step 4: Set ether2 as trunk port

```routeros
/interface bridge port set [find interface=ether2] pvid=10
/interface bridge port set [find interface=ether2] frame-types=admit-all
```

### Step 5: Enable VLAN filtering (only after steps 1–4 are complete)

```routeros
/interface bridge set bridgeLocal vlan-filtering=yes
```

### Step 6: Create DHCP pools and servers for each VLAN

```routeros
# VLAN 10
/ip pool add name=pool-vlan10 ranges=192.168.10.10-192.168.10.254
/ip dhcp-server add name=dhcp-vlan10 interface=LUMINARA address-pool=pool-vlan10 disabled=no
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=8.8.8.8

# VLAN 20
/ip pool add name=pool-vlan20 ranges=192.168.20.10-192.168.20.254
/ip dhcp-server add name=dhcp-vlan20 interface=ONIRICA address-pool=pool-vlan20 disabled=no
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=8.8.8.8
```

## Hyper-V VLAN Assignment

Only one external virtual switch can be bound per physical NIC.

1. Hyper-V Manager → Virtual Switch Manager
2. Remove existing external switch (Network Sync) if present
3. Create new External switch `HomeTrunk-MKR` bound to Realtek adapter
4. In VM settings → Network Adapter → Enable VLAN → set VLAN ID 20

## Recovery: If VLAN Filtering Breaks Connectivity

```routeros
/interface bridge set bridgeLocal vlan-filtering=no
```

Then fix the VLAN table before re-enabling.

## Pending

- [ ] VLAN 30 (VMware) and VLAN 40 (VirtualBox) when needed
- [ ] Inter-VLAN firewall rules (block VLAN 20 from accessing VLAN 10)
- [ ] Dedicated ether3 port for Hyper-V to avoid trunk complexity on ether2
- [ ] Strong admin password
