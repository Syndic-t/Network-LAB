# Incident Report: RB951 Initial Setup — Multiple Issues

**Date:** June 25, 2026
**Device:** MikroTik RB951Ui-2HnD (RouterOS 6.49.7)

## Summary

Six separate issues encountered during initial homelab setup. Documented here for reference and to establish correct procedures.

---

## Issue 1 — ether1 Locked as Bridge Slave

**Error:** `Couldn't change DHCP Client <ether1> — cannot run on slave interface (6)`

**Cause:** Default config places all ports into the bridge. Slave interfaces cannot run DHCP clients.

**Fix:**
1. Bridge → Ports → remove ether1
2. Then add DHCP client on ether1

**Order matters:** Remove from bridge first, then add DHCP client.

---

## Issue 2 — All Connectivity Lost After Removing ether1 from Bridge

**Cause:** Removed ether1 from bridge before the DHCP client was configured, leaving no WAN path.

**Fix:** Always configure the DHCP client on ether1 immediately after removing it from the bridge. Connect the physical cable first.

**Correct order:**
1. Plug ether1 into upstream switch
2. Remove ether1 from bridge
3. Immediately add DHCP client on ether1
4. Confirm WAN IP assigned before continuing

---

## Issue 3 — Factory Reset Wiped All Config

**Cause:** Reset performed with "No Default Configuration" option — produces a completely blank router with no interfaces, IP, DHCP, or WiFi.

**Symptoms:** No WiFi visible. Winbox MAC discovery timed out from both Windows and Ubuntu.

**Recovery:**
1. Perform a new reset using the button on the device — short hold until beep
2. This restores the default config (not blank)
3. Reconnect via Winbox using MAC address

**Rule:** Use "Reset with Default Config" (button beep) for recovery, not "No Default Config" unless you have a prepared blank-slate config ready to push.

---

## Issue 4 — Cannot Set Static IP on Physical Ethernet (Hyper-V)

**Error:** `New-NetIPAddress` and `netsh` both return "Element not found" on the Realtek adapter

**Cause:** Hyper-V creates an external virtual switch (`vEthernet (Network Sync)`) that takes ownership of the Realtek NIC. Windows blocks direct IP assignment to the physical adapter.

**Fix:**
```powershell
# Find the correct interface index
Get-NetAdapter

# Assign IP to the vEthernet adapter, not the physical one
New-NetIPAddress -InterfaceIndex 10 -IPAddress 192.168.88.50 -PrefixLength 24 -DefaultGateway 192.168.88.1
```

**Remember:** On Hyper-V hosts, the vEthernet adapter IS the effective physical ethernet interface for all IP operations.

---

## Issue 5 — DHCP Wizard Rejected Address Pool

**Error:** "Invalid address pool" in DHCP Setup wizard

**Cause:** Wizard UI truncated the input field. Also conflicted with an existing static IP at the pool start address.

**Fix:** Skip the wizard entirely. Use manual setup:

```routeros
/ip pool add name=dhcp-pool ranges=192.168.88.10-192.168.88.254
/ip dhcp-server add name=dhcp1 interface=bridgeLocal address-pool=dhcp-pool disabled=no
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=8.8.8.8
```

---

## Issue 6 — No Internet Despite Correct DHCP, NAT, and Routing

See dedicated incident report: [incident-32-subnet-mask.md](incident-32-subnet-mask.md)

**Short version:** bridgeLocal had `/32` instead of `/24`. Fix: `/ip address add address=192.168.88.1/24 interface=bridgeLocal`

---

## Master Checklist (Post-Incident)

Use this sequence for any fresh MikroTik setup:

- [ ] Connect ether1 physically to upstream
- [ ] Remove ether1 from bridge
- [ ] Add DHCP client on ether1 — confirm IP received
- [ ] Ping 8.8.8.8 from MikroTik (Tools → Ping)
- [ ] Verify `/ip address print` — bridgeLocal shows /24 not /32
- [ ] Create IP pool manually (skip wizard)
- [ ] Create DHCP server and network entry manually
- [ ] Add NAT masquerade on ether1
- [ ] Enable DNS with allow-remote-requests
- [ ] On Hyper-V host: use vEthernet adapter, not physical NIC
- [ ] Test: ping gateway → ping 8.8.8.8 → nslookup google.com
- [ ] Set strong admin password
