# Homelab Wiki

Personal documentation for homelab networking, VM infrastructure, and related projects.

## Structure

```
homelab-wiki/
├── networking/
│   └── mikrotik/
│       ├── concepts/       # Theory and reference material
│       ├── labs/           # Step-by-step configs and setups
│       └── troubleshooting/ # Incident reports and root cause analyses
└── vms/                    # VM infrastructure (coming soon)
```

## Quick Reference

| Topic | Location |
|---|---|
| MikroTik bridge & DHCP concepts | [networking/mikrotik/concepts/](networking/mikrotik/concepts/) |
| RB951 homelab setup | [networking/mikrotik/labs/](networking/mikrotik/labs/) |
| Incident reports | [networking/mikrotik/troubleshooting/](networking/mikrotik/troubleshooting/) |

## Hardware

| Device | Model | Role |
|---|---|---|
| Router | MikroTik RB951Ui-2HnD | Homelab gateway / VLAN router |
| Host PC | Windows 11 | Hyper-V / VMware / VirtualBox host |
| VMs | Ubuntu 24.04, OpenSUSE Leap 16 | Lab servers |
