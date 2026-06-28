# Incident: No Client Internet Access — /32 Subnet Mask on bridgeLocal

**Date:** June 2026
**Device:** MikroTik VM in Hyper-V
**Environment:** RouterOS VM (WAN: 192.168.0.105/24, LAN: bridgeLocal)

## Symptom

Windows host received a valid DHCP lease (192.168.88.253/24, gateway 192.168.88.1) but:

- `ping 192.168.88.1` → Request Timed Out
- `ping 8.8.8.8` → Request Timed Out
- MikroTik itself could ping 8.8.8.8 successfully
- NAT, routing, and DHCP all appeared correct

## Root Cause

The LAN bridge interface was configured with a `/32` host mask instead of `/24`:

```
Incorrect:  192.168.88.1/32   ← single host route only
Correct:    192.168.88.1/24   ← full subnet route
```

With `/32`, RouterOS had no directly connected route for `192.168.88.0/24`. Clients could obtain DHCP leases but the router could not communicate with any host on that subnet.

## Why It's Deceptive

| Service | Status with /32 |
|---|---|
| DHCP server | Working ✓ |
| Client gets valid IP | Yes ✓ |
| Client has correct gateway | Yes ✓ |
| MikroTik internet access | Working ✓ |
| Client ↔ gateway ping | Broken ✗ |
| Client internet access | Broken ✗ |

## Troubleshooting Path (OSI Layer by Layer)

| Layer | Check | Result |
|---|---|---|
| L1 | Interface status | Active |
| L2 | Bridge port membership | ether2 → bridgeLocal ✓ |
| L2 | DHCP lease | 192.168.88.253/24 ✓ |
| L3 | `ping 192.168.88.1` from client | Failed ✗ — identified here |
| L3 | `/ip address print` | Found /32 on bridgeLocal |
| L3 | NAT rule | Present ✓ |
| L3 | IP forwarding | Enabled ✓ |
| L4+ | WAN ping from MikroTik | Working ✓ |

## Resolution

```routeros
/ip address remove 0
/ip address add address=192.168.88.1/24 interface=bridgeLocal
```

Internet access restored immediately.

## Lessons Learned

- Run `/ip address print` early — it's a 5-second check that could save hours
- DHCP functioning correctly does not mean the interface is correctly configured
- Always verify `ping <gateway>` from the client before testing internet access
- A `/32` on a LAN bridge is almost always a configuration error
