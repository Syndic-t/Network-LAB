# NAT and Masquerade

## What NAT Does Here

The MikroTik sits between the homelab subnet (192.168.88.0/24 or VLAN subnets) and the upstream router (192.168.0.0/24). NAT masquerade rewrites outbound traffic so it appears to come from the MikroTik's WAN IP, allowing LAN clients to reach the internet.

## Masquerade Rule

```routeros
/ip firewall nat add chain=srcnat action=masquerade out-interface=ether1
```

This applies to all traffic leaving via ether1 (WAN). The source IP of every packet is replaced with the MikroTik's WAN IP.

## Verify NAT is Present

```routeros
/ip firewall nat print detail
```

Expected output:
```
chain=srcnat action=masquerade out-interface=ether1
```

## Important: NAT Alone is Not Enough

NAT working does not mean clients can reach the internet. The full chain required:

1. Client has a valid IP in the correct subnet
2. Client's default gateway points to the MikroTik LAN IP
3. MikroTik LAN interface has a `/24` address (not `/32`)
4. IP forwarding is enabled (`/ip settings print` → `ip-forward: yes`)
5. NAT masquerade rule exists on ether1
6. MikroTik WAN interface has a valid upstream IP and route
