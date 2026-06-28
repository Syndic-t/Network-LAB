# IP Addressing and Subnet Masks

## The /32 vs /24 Mistake

This is one of the most deceptive misconfigurations in RouterOS because it allows DHCP to work normally while breaking all routing.

### What /32 means

A `/32` mask means the interface owns only that single IP address — it is a host route, not a network route. RouterOS has no knowledge of the surrounding subnet.

```
192.168.88.1/32  →  Only the single IP 192.168.88.1 is "known"
                     No route exists for 192.168.88.0/24
```

### What /24 means

A `/24` mask tells RouterOS that the interface owns the entire `192.168.88.0/24` subnet, creating a directly connected route for it.

```
192.168.88.1/24  →  Route for 192.168.88.0/24 is installed
                     All hosts in .88.x can communicate with the router
```

### Why it's deceptive

With `/32` on the LAN bridge:

- DHCP server still hands out addresses correctly ✓
- Clients get a valid IP, subnet mask, and gateway ✓
- MikroTik can reach the internet itself ✓
- Clients **cannot** ping the gateway ✗
- Clients **cannot** reach the internet ✗

The router literally cannot route for its own LAN subnet.

## How to Check

```routeros
/ip address print
```

Look for the `bridgeLocal` entry. It must show `/24`, not `/32`.

## How to Fix

```routeros
# Remove the incorrect address (check the index number first)
/ip address remove 0

# Add the correct address
/ip address add address=192.168.88.1/24 interface=bridgeLocal
```

## Rule of Thumb

**Always verify `/ip address print` as the first troubleshooting step.** A single wrong mask can break an otherwise perfect configuration.
