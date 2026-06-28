# DHCP Server Setup

## Components Required

A working DHCP server on RouterOS needs three things configured independently:

| Component | What it does |
|---|---|
| IP Pool | Defines the range of addresses to hand out |
| DHCP Server | Binds the pool to an interface |
| DHCP Network | Tells clients their gateway, DNS, and subnet |

## Manual Setup (Preferred Over Wizard)

The DHCP Setup wizard can fail silently — truncating address pool inputs or conflicting with existing static IPs. Manual setup is more reliable:

```routeros
# 1. Create the address pool
/ip pool add name=dhcp-pool ranges=192.168.88.10-192.168.88.254

# 2. Create the DHCP server
/ip dhcp-server add name=dhcp1 interface=bridgeLocal address-pool=dhcp-pool disabled=no

# 3. Create the network entry
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=8.8.8.8
```

## Verify Leases

```routeros
/ip dhcp-server lease print
```

## Important: DHCP Success ≠ Routing Success

A DHCP server distributes addresses based on its own pool configuration — it does not check whether the router interface is correctly configured. A client can receive a valid lease even when the MikroTik's bridge interface has a `/32` mask and cannot route for that subnet.

**Always verify gateway reachability (`ping 192.168.88.1`) after confirming DHCP works.**

## DNS

```routeros
# Set upstream DNS
/ip dns set servers=8.8.8.8

# Allow clients to use the MikroTik as a DNS resolver
/ip dns set allow-remote-requests=yes
```
