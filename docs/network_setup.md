# Network Setup Guide

## Overview

**Goal**: Configure LTE modem → router → optional switch topology with static IP assignments for the main laptop and optional monitoring Pi.

## Hardware Chain

```
[Internet/LTE] → [Netgear LM1200] → [GL.iNet SF1200] → [Homelab Laptop]
                                              └→ [Optional Switch] → [Monitoring Pi]
```

**Network**: `192.168.8.0/24`  
**Router/Gateway**: `192.168.8.1`  
**Infrastructure**: `192.168.8.2-99` (switch, laptop, optional Pi)  
**DHCP Pool**: `192.168.8.100-200` (guests, temporary devices)

## Setup Steps

### 1. LM1200 Modem Setup

1. Insert SIM card
2. Connect to power
3. Connect Ethernet cable from LM1200 LAN port to GL.iNet WAN port
4. Wait for LTE connection (check indicator lights)
5. Connect to Modem on 192.168.5.1
6. Set Settings->Mobile->Auto Connect to "Always" to prevent it from dropping the connection

**Note**: LM1200 should be in bridge/passthrough mode (default) - router handles DHCP/routing.

### 2. GL.iNet SF1200 Router Setup

**Step-by-step workflow:**

1. **Initial setup**: Access `http://192.168.8.1` and set admin password

2. **Generate SSH key** (RSA required for OpenWrt):

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_glinet -C "glinet-router"
```

3. **Configure SSH** (`~/.ssh/config`):

```
Host 192.168.8.1 glinet glinet-router
    HostName 192.168.8.1
    User root
    IdentityFile ~/.ssh/id_rsa_glinet
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedKeyTypes +ssh-rsa
```

4. **Deploy SSH key**:

```bash
ssh-copy-id -i ~/.ssh/id_rsa_glinet -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa root@192.168.8.1
```

5. **Run Ansible setup** (complete pipeline):

```bash
cd ~/Homelab/ansible
cp vars/router_dhcp.yml.example vars/router_dhcp.yml
vim vars/router_dhcp.yml
```

Add the real MAC addresses for the laptop and optional Pi to the vars file.

Optional - set DuckDNS token for DDNS:

```bash
export DUCKDNS_TOKEN="your_token_here"
```

Run the complete setup:

```bash
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml
```

6. **Add VPN client(s)**:

   Edit `playbooks/setup_glinet_wireguard.yml` and add your laptop under `wireguard_clients:`:

   ```yaml
   wireguard_clients:
     - name: laptop
       ip: 10.0.10.2
       description: "Lenovo New"
   ```

   Run the playbook:

   ```bash
   ansible-playbook -i inventory.yml playbooks/setup_glinet_wireguard.yml
   ```

   Get the generated config file from `ansible/wireguard_clients/laptop.conf` and import it into the WireGuard app on your laptop.

   Test VPN connection:

   ```bash
   sudo wg-quick up laptop.conf
   ping 192.168.8.1
   ```

   The ping should reach the router via VPN.

**What gets configured:**

- Python3 installation
- Network and DHCP with static reservations
- DuckDNS
- WireGuard VPN

**Static IP Reservations** (edit `ansible/vars/router_dhcp.yml`):
| Device | MAC Address | Reserved IP |
|-------------------|---------------------|---------------|
| Netgear GS308EP | (from switch label) | 192.168.8.2 |
| Homelab laptop | (from `ip link`) | 192.168.8.10 |
| Monitoring Pi (optional) | (from `ip link`) | 192.168.8.20 |

### 3. Netgear GS308EP Switch Setup (Optional)

1. Connect switch to router (any switch port → router LAN port)
2. Power on switch
3. Access switch web interface at `http://192.168.8.2` (after DHCP reservation takes effect)

**PoE Configuration**:

- Enable PoE only on the port used by the monitoring Pi, if applicable

**Port Assignment** (update `network_inventory.md`):

- Port 1: Uplink to GL.iNet router
- Port 2: Homelab laptop
- Port 3: Monitoring Pi (optional)

### 4. Connect Devices

1. Connect the laptop directly to the router or via the switch
2. Connect the monitoring Pi if you are keeping it
3. Boot each device
4. Verify they receive correct static IPs from DHCP reservations
5. Test connectivity:

```bash
ping 192.168.8.1
ping 8.8.8.8
```
