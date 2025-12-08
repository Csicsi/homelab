# Network Inventory

## Purpose

This document maintains a record of all network-connected devices, their MAC addresses, and IP assignments.

---

## DHCP Reservations

| Device            | Hostname        | MAC Address         | IP Address    | Interface  | Notes                |
| ----------------- | --------------- | ------------------- | ------------- | ---------- | -------------------- |
| ThinkPad T440     | homelab-main    | `XX:XX:XX:XX:XX:XX` | 192.168.8.10  | eth0       | Main server          |
| Asus X550C        | homelab-staging | `XX:XX:XX:XX:XX:XX` | 192.168.8.11  | eth0       | Staging node         |
| MiniPC (Celeron)  | homelab-mgmt    | `XX:XX:XX:XX:XX:XX` | 192.168.8.12  | eth0       | Management/CI-CD/DNS |
| Raspberry Pi 4 #1 | pi4-node1       | `XX:XX:XX:XX:XX:XX` | 192.168.8.20  | eth0       | Critical services    |
| Raspberry Pi 4 #2 | pi4-node2       | `XX:XX:XX:XX:XX:XX` | 192.168.8.21  | eth0       | Critical services    |
| Raspberry Pi 3B+  | pi3-utils       | `XX:XX:XX:XX:XX:XX` | 192.168.8.22  | eth0       | Utilities            |
| Workstation       | workstation     | `XX:XX:XX:XX:XX:XX` | 192.168.8.100 | eth0/wlan0 | Ansible control      |
| Switch            | netgear-gs308ep | `XX:XX:XX:XX:XX:XX` | 192.168.8.2   | mgmt       | PoE+ switch          |

---

## How to Find MAC Addresses

### Linux (Rocky Linux, Ubuntu, Debian)

```bash
ip link show
# or
ip addr show
# Look for "link/ether" line
```

### Raspberry Pi OS

```bash
cat /sys/class/net/eth0/address
# or
ip link show eth0
```

### From Router

- Log into router web interface
- Check DHCP client list / connected devices
- Note MAC and current IP for each device

---

## DNS Records (Future)

When local DNS server is configured (Pi-hole or other), add these records:

| Hostname        | IP           | FQDN                  |
| --------------- | ------------ | --------------------- |
| homelab-main    | 192.168.8.10 | homelab-main.local    |
| homelab-staging | 192.168.8.11 | homelab-staging.local |
| homelab-mgmt    | 192.168.8.12 | homelab-mgmt.local    |
| pi4-node1       | 192.168.8.20 | pi4-node1.local       |
| pi4-node2       | 192.168.8.21 | pi4-node2.local       |
| pi3-utils       | 192.168.8.22 | pi3-utils.local       |

---

## Port Mappings (Switch)

Netgear GS308EP port assignments:

| Port | Connected Device  | PoE Status | Speed     | Notes            |
| ---- | ----------------- | ---------- | --------- | ---------------- |
| 1    | Uplink to router  | Off        | 1000 Mbps | WAN connection   |
| 2    | ThinkPad T440     | Off        | 1000 Mbps | Main server      |
| 3    | Asus X550C        | Off        | 1000 Mbps | Staging          |
| 4    | MiniPC (Celeron)  | Off        | 1000 Mbps | Management node  |
| 5    | Raspberry Pi 4 #1 | On (PoE+)  | 1000 Mbps | Via Evodata hat  |
| 6    | Raspberry Pi 4 #2 | On (PoE+)  | 1000 Mbps | Via Evodata hat  |
| 7    | Raspberry Pi 3B+  | On (PoE)   | 100 Mbps  | Via Evodata hat  |
| 8    | Available         | Off        | -         | Future expansion |

---

## Network Diagram

```
[Internet/LTE]
      |
[Netgear LM1200 Modem]
      |
[GL.iNet SF1200 Router] (192.168.8.1)
      |
[Netgear GS308EP Switch] (192.168.8.2)
      |
      ├── [ThinkPad T440] (192.168.8.10)
      ├── [Asus X550C] (192.168.8.11)
      ├── [MiniPC] (192.168.8.12) - Homer + Pi-hole
      ├── [Pi4 #1] (192.168.8.20)
      └── [Pi4 #2] (192.168.8.21)

[Workstation] Connecting over VPN
```

---

## Notes

- All IPs in range 192.168.8.2-192.168.8.99 are reserved for infrastructure
- DHCP pool for guests/temporary devices: 192.168.8.100-192.168.8.200
