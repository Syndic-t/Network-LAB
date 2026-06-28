# VLANs on RouterOS (Bridge VLAN Filtering)

## Overview

VLANs segment a single physical network into isolated Layer 2 broadcast domains. On RouterOS, VLANs are implemented using bridge VLAN filtering — the bridge acts as a managed switch, enforcing VLAN membership per port.

## Homelab VLAN Layout

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| — | Management | 192.168.88.0/24 | 192.168.88.1 | WiFi & Winbox access |
| 10 | LUMINARA | 192.168.10.0/24 | 192.168.10.1 | Physical PC |
| 20 | ONIRICA | 192.168.20.0/24 | 192.168.20.1 | Hyper-V VMs |

## Port Assignments

| Port | VLAN Mode | VLANs | Connected To |
|---|---|---|---|
| ether1 | WAN | — | Upstream switch |
| ether2 | Trunk | VLAN 10 (untagged) + VLAN 20 (tagged) | PC + Hyper-V |
| ether3 | Access | VLAN 20 (untagged) | Reserved for Hyper-V dedicated port |
| wlan1 | Management | Untagged | WiFi management |

## Critical Rule: Populate VLAN Table Before Enabling Filtering

Enabling VLAN filtering on an incomplete VLAN table drops all traffic immediately. Always:

1. Create all VLAN entries first
2. Verify the table is correct
3. Then enable filtering

## Creating VLANs — Use Terminal, Not Winbox UI

The Winbox UI creates **dynamic** VLAN entries (shown with a `D` flag) which are read-only and cannot be edited. Always use terminal:

```routeros
# Create VLAN 10 — ether2 as untagged access port
/interface bridge vlan add bridge=bridgeLocal vlan-ids=10 tagged=bridgeLocal untagged=ether2

# Create VLAN 20 — ether2 as tagged trunk, ether3 as untagged access
/interface bridge vlan add bridge=bridgeLocal vlan-ids=20 tagged=bridgeLocal,ether2 untagged=ether3
```

## Enable VLAN Filtering

```routeros
/interface bridge set bridgeLocal vlan-filtering=yes
```

## Disable VLAN Filtering (Recovery)

If you lose connectivity after enabling filtering:

```routeros
/interface bridge set bridgeLocal vlan-filtering=no
```

## VLAN Interfaces and Gateway IPs

Each VLAN needs a corresponding virtual interface and a manually assigned gateway IP. They do not self-assign `.1` addresses.

```routeros
# Create VLAN interfaces
/interface vlan add name=LUMINARA vlan-id=10 interface=bridgeLocal
/interface vlan add name=ONIRICA vlan-id=20 interface=bridgeLocal

# Assign gateway IPs
/ip address add address=192.168.10.1/24 interface=LUMINARA
/ip address add address=192.168.20.1/24 interface=ONIRICA
```

## Set PVID on Trunk Port

```routeros
# ether2 native VLAN is VLAN 10 (PC traffic arrives untagged)
/interface bridge port set [find interface=ether2] pvid=10

# Allow both tagged and untagged frames on ether2
/interface bridge port set [find interface=ether2] frame-types=admit-all
```

## Hyper-V VLAN Assignment

Hyper-V VMs are placed on a VLAN by:

1. Creating an External Virtual Switch bound to the physical NIC (only one switch per NIC)
2. Assigning the VM's network adapter a VLAN ID in Hyper-V settings

Only one Hyper-V external virtual switch can be bound to a single physical adapter. Remove existing switches before creating a new one on the same adapter.
