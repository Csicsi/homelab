# Network Inventory

## Purpose

This document tracks the simplified homelab network: one main laptop, one router, and an optional Raspberry Pi for redundancy.

---

## DHCP Reservations

| Device         | Hostname        | MAC Address         | IP Address    | Interface  | Notes                            |
| -------------- | --------------- | ------------------- | ------------- | ---------- | -------------------------------- |
| Homelab laptop | homelab-laptop  | `XX:XX:XX:XX:XX:XX` | 192.168.8.10  | eth0/wlan0 | Primary host and Ansible control |
| Monitoring Pi  | monitoring-pi   | `XX:XX:XX:XX:XX:XX` | 192.168.8.20  | eth0       | Optional redundancy and alerting |
| Workstation    | workstation     | `XX:XX:XX:XX:XX:XX` | 192.168.8.100 | eth0/wlan0 | Optional separate admin machine  |
| Switch         | netgear-gs308ep | `XX:XX:XX:XX:XX:XX` | 192.168.8.2   | mgmt       | Optional switch                  |

---

## How to Find MAC Addresses

### Linux (Ubuntu, Debian, Raspberry Pi OS)

```bash
ip link show
# or
ip addr show
```

For a Pi ethernet MAC specifically:

```bash
cat /sys/class/net/eth0/address
```

### From Router

- Log into the router web interface
- Check the DHCP client list or connected devices
- Record the MAC and current IP

---

## DNS Records (Future)

If local DNS is added with Pi-hole or another resolver, start with these names:

| Hostname       | IP           | FQDN                 |
| -------------- | ------------ | -------------------- |
| homelab-laptop | 192.168.8.10 | homelab-laptop.local |
| monitoring-pi  | 192.168.8.20 | monitoring-pi.local  |

---

## Port Mappings (Optional Switch)

Netgear GS308EP suggested assignments:

| Port | Connected Device | PoE Status | Speed     | Notes            |
| ---- | ---------------- | ---------- | --------- | ---------------- |
| 1    | Uplink to router | Off        | 1000 Mbps | LAN uplink       |
| 2    | Homelab laptop   | Off        | 1000 Mbps | Primary host     |
| 3    | Monitoring Pi    | On (PoE)   | 100/1000  | Optional node    |
| 4    | Available        | Off        | -         | Future expansion |
| 5    | Available        | Off        | -         | Future expansion |
| 6    | Available        | Off        | -         | Future expansion |
| 7    | Available        | Off        | -         | Future expansion |
| 8    | Available        | Off        | -         | Future expansion |

---

## Network Diagram

```
[Internet/LTE]
      |
[Netgear LM1200 Modem]
      |
[GL.iNet SF1200 Router] (192.168.8.1)
      |
      ├── [Homelab Laptop] (192.168.8.10)
      └── [Optional Switch] (192.168.8.2)
             └── [Monitoring Pi] (192.168.8.20)

[Workstation] Connecting over VPN
```

---

## Notes

- All IPs in range 192.168.8.2-192.168.8.99 are reserved for infrastructure
- Only the router, laptop, optional Pi, and switch need static reservations by default
- DHCP pool for guests and temporary devices: 192.168.8.100-192.168.8.200
