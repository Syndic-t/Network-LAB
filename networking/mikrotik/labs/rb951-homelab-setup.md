# Lab: RB951Ui-2HnD Homelab Router Setup

**Date:** June 25, 2026
**Device:** MikroTik RB951Ui-2HnD
**RouterOS:** 6.49.7
**Tool:** Winbox 4.1

## Objective

Deploy the MikroTik as a secondary router behind an existing home router, creating an isolated homelab network for a Windows 11 PC and its virtual machines (Hyper-V, VMware, VirtualBox).

## Network Design

```
Internet
    │
192.168.0.1  (Main Router)
    │
Unmanaged Switch
    │
ether1 ──── MikroTik RB951 ──── ether2–5, wlan1
            192.168.0.105/24         │
            (WAN/DHCP from           bridgeLocal
             main router)            192.168.88.1/24
                                     │
                              PC + VMs (192.168.88.x)
```

## Final Configuration

### Interfaces

| Interface | Role | IP |
|---|---|---|
| ether1 | WAN | 192.168.0.105/24 (DHCP from main router) |
| bridgeLocal | LAN | 192.168.88.1/24 |
| wlan1 | WiFi (management) | Member of bridgeLocal |

### DHCP

- Pool: `dhcp-pool` → 192.168.88.10–192.168.88.254
- Server: on `bridgeLocal`
- Network: 192.168.88.0/24, gateway 192.168.88.1, DNS 8.8.8.8

### NAT

```routeros
/ip firewall nat add chain=srcnat action=masquerade out-interface=ether1
```

### DNS

```routeros
/ip dns set servers=8.8.8.8 allow-remote-requests=yes
```

## Setup Order (Correct Sequence)

This order matters — deviating from it causes connectivity loss:

1. Connect ether1 physically to the unmanaged switch
2. Remove ether1 from bridgeLocal port list
3. Add DHCP client on ether1 — confirm it gets an IP from main router
4. Verify MikroTik can ping 8.8.8.8
5. Set static IP on bridgeLocal: `192.168.88.1/24` (not /32)
6. Create IP pool manually
7. Create DHCP server manually
8. Create DHCP network entry
9. Add NAT masquerade rule on ether1
10. Enable DNS, set upstream to 8.8.8.8
11. Test from client: ping gateway → ping 8.8.8.8 → ping google.com

## Hyper-V Note

On Windows 11 with Hyper-V, the physical Realtek NIC is owned by the Hyper-V virtual switch (`vEthernet (Network Sync)`). You cannot assign IPs directly to the physical adapter.

- Use `Get-NetAdapter` to identify the vEthernet interface index
- Assign IPs to `vEthernet (Network Sync)` not to the Realtek adapter
- PowerShell: `New-NetIPAddress -InterfaceIndex 10 -IPAddress 192.168.88.50 -PrefixLength 24 -DefaultGateway 192.168.88.1`

## Pending

- [ ] Strong admin password: System → Users → admin
- [ ] VLAN setup for VM isolation (see [vlan-setup lab](vlan-setup.md))
- [ ] Firewall rules to block 192.168.88.x → 192.168.0.x
