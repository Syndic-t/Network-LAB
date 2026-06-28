# Incident Report: SSH Access Failure After MikroTik VLAN Migration

**Date:** June 28, 2026
**Environment:** Windows 11 Host, Hyper-V VM (Ubuntu), MikroTik RB951Ui-2HnD
**Status:** Resolved

---

## 1. Overview

Following the successful deployment of VLAN segmentation on the MikroTik router, SSH access to a Hyper-V hosted Ubuntu VM stopped working. The VM had previously been accessible but became unreachable after being migrated onto a dedicated VM VLAN.

---

## 2. Environment

| Component | Detail |
|---|---|
| Host OS | Windows 11 with Hyper-V |
| VM OS | Ubuntu (Hyper-V guest) |
| Router | MikroTik RB951Ui-2HnD |
| PC VLAN | LUMINARA — physical PC segment |
| VM VLAN | ONIRICA — Hyper-V VM segment |
| Hypervisor Virtual Switch | HomeTrunk-MKR (external, trunk mode) |

---

## 3. Symptoms

- SSH connection attempts to the VM resulted in **connection refused** or **connection timed out**
- The VM could ping the MikroTik gateway successfully
- The Windows PC could not ping the VM at all
- SSH service was confirmed running on the VM
- SSH was configured on a non-standard port

---

## 4. Troubleshooting Path

### Step 1 — Verify SSH service

Checked SSH service status on the VM via Hyper-V console:

```bash
systemctl status ssh
```

Result: Service active and running. Ruled out SSH daemon as the cause.

### Step 2 — Verify SSH listening port

```bash
grep Port /etc/ssh/sshd_config
```

Confirmed SSH was listening on the correct non-standard port. Ruled out port mismatch.

### Step 3 — Test network reachability

Ran verbose SSH from the Windows host:

```bash
ssh -v user@<vm-ip>
```

Result: **Connection refused** — confirmed the packet was reaching the VM's network layer but being rejected.

### Step 4 — Check MikroTik NAT rules

Attempted to add a DST-NAT port forwarding rule on MikroTik to forward SSH traffic to the VM. Initial rule used `ether1` (WAN) as the inbound interface.

Result: Still timed out. Realised the traffic was internal (PC to VM, same router) and never traverses ether1.

### Step 5 — Identify VLAN isolation as root cause

Attempted ping from Windows PC to VM — timed out. This confirmed the real issue: **the PC and VM were on separate VLANs with no inter-VLAN routing permitted**, so traffic between them was being dropped at the bridge level before even reaching SSH.

### Step 6 — Add inter-VLAN firewall filter rules

Added forward chain rules on the MikroTik to permit traffic between the PC VLAN and VM VLAN:

```routeros
/ip firewall filter add chain=forward action=accept \
    src-address=<pc-vlan-subnet> dst-address=<vm-vlan-subnet>

/ip firewall filter add chain=forward action=accept \
    src-address=<vm-vlan-subnet> dst-address=<pc-vlan-subnet>
```

Result: Ping still timed out. Rules were correct but the problem persisted.

### Step 7 — Verify routing table

```routeros
/ip route print
```

Both VLAN subnets showed as active directly connected routes. Routing was not the issue.

### Step 8 — Verify IP forwarding

```routeros
/ip settings print
```

`ip-forward: yes` confirmed. Not the issue.

### Step 9 — Inspect VM network configuration

```bash
ip addr show
ip route show
```

**Root cause identified:** The VM's single ethernet interface (`eth0`) had **two IP addresses assigned** — one from each VLAN subnet. It had also received two default gateway entries, one per VLAN. This caused:

- Ambiguous routing on the VM (two defaults, two subnets on one interface)
- The VM responding on the wrong subnet depending on which route was preferred
- Inconsistent reachability from the PC

---

## 5. Root Cause

During the VM's network configuration, VLAN 10 (PC VLAN) routing details were manually added to the VM's network configuration and never removed when VLAN 20 (VM VLAN) was subsequently configured. This left the VM with:

- Two IP addresses on `eth0` — one from each VLAN subnet
- Two default gateway entries — one per VLAN

The duplicate IP and duplicate default gateway caused routing ambiguity that made the VM unreachable in a consistent manner. This was a manual configuration error, not a DHCP or trunk port issue.

---

## 6. Resolution

Removed the incorrect VLAN address and its associated default gateway from the VM:

```bash
sudo ip addr del <incorrect-ip>/24 dev eth0
sudo ip route del default via <incorrect-gateway>
```

Made the change permanent by updating the Netplan configuration to statically define only the correct VLAN address and gateway, preventing the issue from recurring on reboot.

After cleanup:
- VM retained only its correct VLAN IP
- Single default gateway pointed to the correct VLAN gateway
- PC could ping the VM successfully
- SSH connection established successfully

---

## 7. Were the Firewall Filter Rules Necessary?

Technically no — they were not the fix. Since both VLANs are directly connected routes on the MikroTik, inter-VLAN routing works by default without explicit filter rules.

However the rules are **worth keeping** as they form the foundation of inter-VLAN access control. The next step is to tighten them — rather than allowing full bidirectional traffic, restrict VM-initiated connections back to the PC to established/related traffic only:

```routeros
# Allow PC to initiate connections to VMs
chain=forward action=accept src=<pc-vlan> dst=<vm-vlan>

# Allow only return traffic from VMs (not new connections)
chain=forward action=accept src=<vm-vlan> dst=<pc-vlan> \
    connection-state=established,related

# Drop VM-initiated connections to PC
chain=forward action=drop src=<vm-vlan> dst=<pc-vlan>
```

---

## 8. Lessons Learned

- **Clean up old network config when migrating a VM between VLANs** — when moving a VM from one VLAN to another, explicitly remove the old VLAN's IP and gateway entries. Leaving stale config in place causes dual-IP situations that produce unpredictable routing behaviour.

- **Multiple IPs on one interface causes unpredictable routing** — two default gateways on the same interface creates metric-based ambiguity. Always verify `ip addr show` and `ip route show` after any network change on a VM.

- **Connection refused vs timed out means different things:**
  - `Connection refused` — packet reached the host, port rejected it (service down or wrong port)
  - `Connection timed out` — packet never reached the host (firewall, routing, or VLAN issue)

- **Firewall rules alone don't fix routing** — if inter-VLAN traffic isn't flowing, check the VM's own IP configuration before assuming the router is at fault.

- **Always test with ping before SSH** — a simple ping between subnets immediately reveals whether the network layer is working, saving time chasing application-layer issues.

- **Non-standard SSH ports require explicit port forwarding rules** — document the port in your config management so it isn't forgotten during troubleshooting.

---

## 9. Verification Checklist

Use after any VLAN or network change affecting a VM:

- [ ] `ip addr show` — VM has only one IP, on the correct subnet
- [ ] `ip route show` — single default gateway, correct VLAN gateway IP
- [ ] `ping <gateway>` from VM — succeeds
- [ ] `ping <vm-ip>` from PC — succeeds
- [ ] `ssh -p <port> user@<vm-ip>` from PC — connects successfully
- [ ] Netplan or network config updated to make changes permanent
