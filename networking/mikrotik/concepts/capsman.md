# CAPsMAN vs Standalone WiFi

## What is CAPsMAN?

CAPsMAN (Controlled Access Point system Manager) is MikroTik's centralised wireless management system. It allows one RouterOS device to manage the wireless configuration of multiple access points.

On the RB951Ui-2HnD, the default config enables CAPsMAN and assigns wlan1 to it — even on a single-device setup where it adds no value.

## The Problem

When wlan1 is managed by CAPsMAN, it is locked as a slave interface. Any attempt to configure it directly returns:

```
managed by CAPsMAN
```

Disabling CAPsMAN through the Winbox UI alone does not release wlan1.

## How to Release wlan1 from CAPsMAN

Must be done via terminal:

```routeros
/interface wireless cap set enabled=no interfaces=""
/caps-man manager set enabled=no
/caps-man provisioning remove [find]
/caps-man manager interface set [find default=yes] forbid=no
```

After running these commands, the red CAPsMAN banner disappears and wlan1 can be configured normally.

## Standalone AP Configuration

```routeros
# Enable the interface
/interface wireless enable wlan1

# Set mode and radio settings
/interface wireless set wlan1 mode=ap-bridge band=2ghz-b/g/n frequency=auto ssid=YourSSID

# Create WPA2 security profile (via Winbox: Wireless → Security Profiles)
# Then assign it to wlan1
/interface wireless set wlan1 security-profile=default
```

## When to Use CAPsMAN

Only useful when managing multiple physical access points from a single controller. For a single RB951 homelab setup, disable it and use standalone AP mode.
