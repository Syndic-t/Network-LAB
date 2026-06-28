# Bridges and Bridge Ports

## What is a Bridge?

A bridge in RouterOS combines multiple interfaces into a single Layer 2 broadcast domain — similar to a software switch. All ports added to the bridge can communicate with each other at Layer 2 without routing.

## Default Config Behaviour

Out of the box, the RB951Ui-2HnD places **all ports** (ether1–ether5 and wlan1) into `bridgeLocal` as slave interfaces. This is fine for a simple home router but must be changed when separating WAN from LAN.

## WAN vs LAN Port Separation

`ether1` must be **removed** from the bridge to act as a standalone WAN port:

1. Bridge → Ports tab → select ether1 → remove (minus button)
2. Then assign a DHCP Client to ether1 via IP → DHCP Client

> **Important:** Always configure the DHCP client on ether1 BEFORE removing it from the bridge, or you will lose all connectivity.

## Slave Interface Limitation

A bridge slave interface cannot run a DHCP client directly. Attempting it produces:

```
Error: Couldn't change DHCP Client <ether1> — cannot run on slave interface (6)
```

The fix is always to remove the interface from the bridge first.

## Bridge Port Roles

| Port | Role | Notes |
|---|---|---|
| ether1 | WAN | Standalone, DHCP client from upstream router |
| ether2–ether5 | LAN | Bridge slaves, carry PC and VM traffic |
| wlan1 | Management / LAN | Bridge slave, WiFi access point |

## Key Commands

```routeros
# View bridge ports
/interface bridge port print

# Remove a port from the bridge
/interface bridge port remove [find interface=ether1]

# Add a port to the bridge
/interface bridge port add interface=ether2 bridge=bridgeLocal
```
