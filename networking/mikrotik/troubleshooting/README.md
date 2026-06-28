# MikroTik Troubleshooting

Incident reports and root cause analyses from real issues encountered.

| Incident | Root Cause | Impact |
|---|---|---|
| [/32 subnet mask on bridgeLocal](incident-32-subnet-mask.md) | Wrong prefix length on LAN bridge | Clients had DHCP but no internet |
| [RB951 initial setup — 6 issues](incident-rb951-setup.md) | Bridge config, Hyper-V NIC, DHCP wizard, factory reset | Full setup session |
| [WiFi & VLAN setup issues](incident-wifi-vlan-setup.md) | CAPsMAN lock, dynamic VLANs, Hyper-V switch conflict | Session 2 |

## Quick Lookup

**Client has IP but can't reach gateway** → Check `/ip address print` for `/32` on bridge interface

**Can't configure ether1 DHCP** → Remove ether1 from bridge first

**VLAN filtering dropped all traffic** → VLAN table was incomplete; disable filtering, fix table, re-enable

**wlan1 shows "managed by CAPsMAN"** → Must use terminal commands to release (UI is not enough)

**Can't set IP on Windows ethernet** → Hyper-V owns the NIC; use the `vEthernet` adapter instead

**Winbox MAC discovery times out** → Router may have been reset with "No Default Config"; reset again with button beep
