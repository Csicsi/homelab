# Ansible Playbooks

## Overview

Playbooks are organized by target system and purpose. Router playbooks can be run individually for updates, or use the complete pipeline for initial setup.

---

## Router Setup Pipeline

### `setup_glinet_complete.yml` ⭐ **Recommended for Initial Setup**

**Target**: `glinet-router` (GL.iNet SF1200)

**Purpose**: Complete router setup pipeline - orchestrates all router playbooks in correct order

**What it does**:

- Runs all router playbooks in sequence:
  1. Bootstrap (Python installation)
  2. Network/DHCP configuration
  3. DuckDNS setup (optional, prompted)
  4. WireGuard VPN (optional, prompted)
- Provides interactive prompts to skip optional components
- Shows progress and completion summary

**Prerequisites**:

- Router accessible via SSH with RSA key
- `ansible/vars/router_dhcp.yml` configured with MAC addresses
- `DUCKDNS_TOKEN` environment variable (if using DDNS)
- Edit WireGuard endpoint in playbook vars (if using VPN)

**Usage**:

```bash
# Set DuckDNS token if using dynamic DNS
export DUCKDNS_TOKEN="your-token-from-duckdns-org"

# Run complete pipeline
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml

# Or run without prompts (all optional features enabled)
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml -e "setup_ddns=yes" -e "setup_vpn=yes"
```

**What gets configured**:

- ✅ Python3 for Ansible
- ✅ Static DHCP reservations for all homelab devices
- ✅ DuckDNS for dynamic public IP (if enabled)
- ✅ WireGuard VPN server with client configs (if enabled)

**Note**: Individual playbooks can still be run separately for updates/maintenance.

---

## Individual Router Playbooks

### `bootstrap_glinet.yml`

**Target**: `glinet-router` (GL.iNet SF1200)

**Purpose**: Initial bootstrap - installs Python on router for Ansible to work

**What it does**:

- Checks if Python is already installed
- Updates opkg package list
- Installs Python 3 (required for Ansible modules)
- Verifies installation and tests Ansible connectivity

**Prerequisites**:

- Router accessible via SSH with RSA key
- Internet connection on router

**Usage**:

```bash
# Run immediately after SSH setup
ansible-playbook -i inventory.yml playbooks/bootstrap_glinet.yml
```

**Tags**: `bootstrap`, `test`

**Note**: This playbook uses `raw` commands (doesn't require Python), so it can run before Python is installed.

---

### `setup_glinet_openwrt.yml`

**Target**: `glinet-router` (GL.iNet SF1200)

**Purpose**: Configure GL.iNet router network settings and DHCP reservations via UCI

**What it does**:

- Sets LAN IP address and netmask
- Configures DHCP server range and lease time
- Adds static DHCP reservations for all homelab devices
- Installs useful network tools (htop, tcpdump, iperf3)

**Prerequisites**:

- Router has stock GL.iNet firmware (default)
- Root password set manually via SSH
- **RSA SSH key generated and deployed** (router requires RSA, not ED25519):
  ```bash
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_glinet -C "glinet-router"
  ssh-copy-id -i ~/.ssh/id_rsa_glinet -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa root@192.168.8.1
  ```
- SSH access working (test: `ansible -i inventory.yml glinet-router -m ping`)
- **Copy vars file**: `cp ../vars/router_dhcp.yml.example ../vars/router_dhcp.yml`
- **Update MAC addresses** in `../vars/router_dhcp.yml` with actual device MACs

**Usage**:

```bash
# 1. Copy and edit vars file
cp ansible/vars/router_dhcp.yml.example ansible/vars/router_dhcp.yml
vim ansible/vars/router_dhcp.yml  # Add real MAC addresses

# 2. Run playbook
ansible-playbook -i inventory.yml playbooks/setup_glinet_openwrt.yml
```

**Tags**: `network`, `dhcp`, `packages`

**Note**: See `docs/network_setup.md` for initial router setup steps and `docs/ansible.md` for SSH configuration.

---

### `setup_glinet_ddns.yml`

**Target**: `glinet-router` (GL.iNet SF1200)

**Purpose**: Configure DuckDNS dynamic DNS service for remote access with changing public IP

**What it does**:

- Installs DDNS packages (ddns-scripts, ddns-scripts-services, luci-app-ddns)
- Configures DuckDNS service with domain and token
- Sets update intervals (checks every 10 minutes, forces every 24 hours)
- Enables and starts DDNS service
- Verifies DNS resolution matches current public IP

**Prerequisites**:

- Router has internet connection
- DuckDNS account and token: https://www.duckdns.org
- Environment variable `DUCKDNS_TOKEN` set or edit playbook vars

**Usage**:

```bash
# Set your DuckDNS token
export DUCKDNS_TOKEN="your-token-from-duckdns-org"

# Run playbook
ansible-playbook -i inventory.yml playbooks/setup_glinet_ddns.yml
```

**Tags**: `packages`, `ddns`

**Configuration**:

Edit playbook vars to customize:

- `duckdns_domain`: Your subdomain (e.g., "yourname" for yourname.duckdns.org)
- `ddns_check_interval`: How often to check for IP changes (default 600s = 10min)
- `ddns_force_interval`: Force update even if IP unchanged (default 86400s = 24h)

**Note**: Required for WireGuard VPN if you don't have a static public IP. The VPN endpoint will be `yourdomain.duckdns.org`.

---

### `setup_glinet_wireguard.yml`

**Target**: `glinet-router` (GL.iNet SF1200)

**Purpose**: Configure WireGuard VPN server on router for secure remote access to homelab

**What it does**:

- Installs WireGuard packages (wireguard-tools, kmod-wireguard)
- Generates server private/public key pair
- Creates WireGuard network interface (wg0)
- Configures firewall zones and rules
- Opens UDP port 51820 from WAN
- Allows VPN clients to access LAN

**Prerequisites**:

- Basic router setup completed (`setup_glinet_openwrt.yml`)
- Public IP or DDNS configured
- UDP port 51820 forwarded on LM1200 modem

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_glinet_wireguard.yml
```

**Tags**: `packages`, `wireguard`, `firewall`

**Post-setup**:

1. Note the server public key from playbook output
2. Create client configs via GL.iNet web UI or manually
3. Test VPN connection from remote location

**Note**: See `docs/network_setup.md` for WireGuard setup details and client configuration.

---

## Raspberry Pi Playbooks

### `setup_pis.yml`

**Target**: `pis` (all Raspberry Pis - pi4-node1, pi4-node2, pi3-utils)

**Purpose**: Post-NVMe migration setup - essential packages, Docker, firewall, k3s preparation

**What it does**:

- Updates all system packages
- Installs essential tools (vim, git, htop, python3, etc.)
- Verifies root filesystem is on NVMe (/dev/sda)
- Installs Docker and Docker Compose
- Enables memory cgroup for k3s support
- Configures UFW firewall (SSH, k3s ports for Pi4s)
- Performance tuning (swap, inotify limits)
- SSH hardening (disable password auth, disable root login)
- Sets timezone to America/New_York

**Prerequisites**:

- Pis migrated to NVMe using `docs/pi_nvme_migration.md` process
- SSH key deployed: `ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@<pi-ip>`
- Pis accessible via SSH (test: `ansible -i inventory.yml pis -m ping`)

**Usage**:

```bash
# Test connectivity first
ansible -i inventory.yml pis -m ping

# Run complete setup
ansible-playbook -i inventory.yml playbooks/setup_pis.yml --ask-become-pass

# Run specific sections
ansible-playbook -i inventory.yml playbooks/setup_pis.yml --tags docker --ask-become-pass
ansible-playbook -i inventory.yml playbooks/setup_pis.yml --tags firewall --ask-become-pass
```

**Tags**: `update`, `packages`, `system`, `verify`, `docker`, `firewall`, `performance`, `ssh`

**Post-setup**:

- Log out and back in for docker group changes to take effect
- Verify NVMe boot: `lsblk -f` should show root (/) on sda2
- Next step: Install k3s cluster on Pi4 nodes

---

## Homelab Main Server Playbooks

### `setup_servers.yml`

**Target**: `servers` (homelab-main, homelab-staging)

**Purpose**: Initial setup of x86 servers with Docker, firewall, and laptop-specific configs

**What it does**:

- Updates all system packages (apt-based)
- Installs essential packages (vim, git, curl, etc.)
- Adds Docker repository and installs Docker + Docker Compose
- Configures laptop to stay running with lid closed
- Sets up UFW firewall (SSH, HTTP, HTTPS)
- Installs and enables Nginx reverse proxy
- Adds user to docker group for non-root Docker access

**Prerequisites**:

- Ubuntu Server 24.04 LTS installed on target hosts
- SSH keys deployed to all servers:
  - `ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@192.168.8.10`
  - `ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@192.168.8.11`
- Internet connection for package downloads

**Usage**:

```bash
# Test connectivity
ansible -i inventory.yml servers -m ping

# Run complete setup on all servers
ansible-playbook -i inventory.yml playbooks/setup_servers.yml --ask-become-pass

# Run on specific server only
ansible-playbook -i inventory.yml playbooks/setup_servers.yml --ask-become-pass --limit homelab-staging

# Run specific sections
ansible-playbook -i inventory.yml playbooks/setup_servers.yml --tags docker --ask-become-pass
ansible-playbook -i inventory.yml playbooks/setup_servers.yml --tags firewall --ask-become-pass
```

**Tags**: `update`, `packages`, `docker`, `nginx`, `firewall`, `lid`

**Post-setup**:

- Log out and back in for docker group changes: `newgrp docker`
- Verify Docker: `docker ps`
- Check firewall status: `sudo ufw status`
- Test Nginx: `curl http://localhost`

---

### `setup_main_server_jenkins.yml`

**Target**: `homelab-main` (ThinkPad T440)

**Purpose**: Deploy Jenkins CI/CD server in Docker, open firewall port, and prepare persistent storage

**What it does**:

- Pulls Jenkins Docker image (`jenkins/jenkins:lts`)
- Creates persistent Jenkins data directory (`/srv/jenkins`)
- Runs Jenkins container on port 9080 (mapped to 8080 in container)
- Sets restart policy to always
- Opens port 9080/tcp in firewalld

**Prerequisites**:

- Docker and firewalld installed and running (see `setup_main_server.yml`)
- Python3-requests installed on target host (included in essentials)
- Ansible `community.docker` collection installed on control node (`ansible-galaxy collection install community.docker`)

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_main_server_jenkins.yml --ask-become-pass
```

**Tags**: `jenkins`, `firewall`

**Notes**:

- Retrieve initial admin password: `sudo cat /srv/jenkins/secrets/initialAdminPassword`
- Jenkins UI accessible: `http://192.168.8.10:9080`
- Next step: Configure Jenkins for GitHub integration

---

## Management Node Service Playbooks

### `setup_jenkins.yml`

**Target**: `homelab-mgmt` (192.168.8.12)

**Purpose**: Deploy Jenkins CI/CD server

**What it does**:

- Creates Jenkins data directory in `/srv/jenkins`
- Deploys Jenkins container with Docker-in-Docker support
- Configures ports 9080 (web) and 50000 (agents)
- Opens firewall ports
- Displays initial admin password

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml --ask-become-pass
```

**Access**: http://192.168.8.12:9080/jenkins

---

### `setup_portainer.yml`

**Target**: `homelab-mgmt` (192.168.8.12)

**Purpose**: Deploy Portainer container management UI

**What it does**:

- Creates Portainer data directory in `/srv/portainer`
- Deploys Portainer CE container
- Configures ports 9000 (HTTP) and 9443 (HTTPS)
- Opens firewall ports

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_portainer.yml --ask-become-pass
```

**Access**: https://192.168.8.12:9443 (create admin password on first login)

---

### `setup_homer.yml`

**Target**: `homelab-mgmt` (192.168.8.12)

**Purpose**: Deploy Homer dashboard with service links

**What it does**:

- Creates Homer configuration with all homelab services
- Deploys Homer container on port 8081
- Configures firewall
- Provides unified dashboard for all services

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_homer.yml --ask-become-pass
```

**Access**: http://192.168.8.12:8081

---

### `setup_pihole.yml`

**Target**: `homelab-mgmt` (192.168.8.12)

**Purpose**: Deploy Pi-hole DNS server and ad blocker

**What it does**:

- Creates Pi-hole directories in `/opt/pihole`
- Deploys Pi-hole container
- Stops systemd-resolved (conflicts with DNS)
- Configures DNS ports and firewall
- Sets up upstream DNS servers

**Variables** (edit in playbook):

- `pihole_webpassword`: Admin password (default: "admin")
- `pihole_timezone`: Timezone (default: "Europe/Vienna")
- `pihole_dns1/dns2`: Upstream DNS servers (default: Cloudflare)

**Usage**:

```bash
ansible-playbook -i inventory.yml playbooks/setup_pihole.yml --ask-become-pass
```

**Access**: http://192.168.8.12:8080/admin

**Post-setup**: Update router DNS to 192.168.8.12 for network-wide ad blocking

---

## Kubernetes and Monitoring Playbooks

### `setup_k3s_cluster.yml`

**Target**: `pis` group (specifically pi4-node1 and pi4-node2)

**Purpose**: Install and configure k3s Kubernetes cluster on Pi4 nodes

**What it does**:

- Installs k3s server on pi4-node1 (192.168.8.20)
- Installs k3s agent on pi4-node2 (192.168.8.21)
- Disables Traefik and ServiceLB (not needed for simple setup)
- Configures kubeconfig for cluster access
- Verifies cluster nodes are ready

**Prerequisites**:

- Pis configured with `setup_pis.yml`
- Memory cgroups enabled in `/boot/firmware/cmdline.txt`
- Firewall allows k3s ports (6443, 10250)

**Usage**:

```bash
# Install k3s cluster
ansible-playbook -i inventory.yml playbooks/setup_k3s_cluster.yml --ask-become-pass

# Verify cluster from pi4-node1
ssh dcsicsak@192.168.8.20
kubectl get nodes
```

**Tags**: `install`, `token`, `kubeconfig`, `verify`

**Post-setup**:

- Access cluster: `ssh dcsicsak@192.168.8.20`, then use `kubectl`
- Node token saved to `/tmp/k3s-node-token` on control machine
- Kubeconfig at `~/.kube/config` on pi4-node1

---

### `setup_monitoring_stack.yml`

**Target**: `pi4-node1` (for k3s deployments) and `all` (for node exporters)

**Purpose**: Deploy complete monitoring stack (Prometheus, Grafana, Loki) to k3s cluster

**What it does**:

- Deploys Prometheus to k3s with scrape configs for all hosts
- Deploys Grafana with admin/admin credentials
- Deploys Loki for log aggregation
- Installs node_exporter on all hosts (systemd service)
- Configures firewalls for metrics collection (port 9100)
- Uses NodePort services for external access

**Prerequisites**:

- k3s cluster running (`setup_k3s_cluster.yml`)
- Docker installed on all hosts
- Internet access for downloading components

**Usage**:

```bash
# Deploy complete monitoring stack
ansible-playbook -i inventory.yml playbooks/setup_monitoring_stack.yml --ask-become-pass
```

**Tags**: `namespace`, `manifests`, `prometheus`, `loki`, `grafana`, `verify`, `node-exporter`

**Access URLs**:

- Prometheus: http://192.168.8.20:30090
- Grafana: http://192.168.8.20:30030 (admin/admin)

**Post-setup**:

1. Open Grafana in browser
2. Add Prometheus data source: `http://prometheus.monitoring.svc.cluster.local:9090`
3. Add Loki data source: `http://loki.monitoring.svc.cluster.local:3100`
4. Import dashboards (Node Exporter Full, Kubernetes Cluster Monitoring)

---

### `setup_pihole.yml`

**Target**: `homelab-mgmt` (192.168.8.12)

**Purpose**: Deploy Pi-hole DNS ad blocker via Docker Compose on Management Node

**What it does**:

- Creates Pi-hole directories (`/opt/pihole`)
- Deploys Pi-hole container via docker-compose
- Stops systemd-resolved (conflicts with Pi-hole DNS)
- Configures static resolv.conf
- Opens firewall ports (53 for DNS, 8080 for web)

**Prerequisites**:

- Docker and Docker Compose installed (`setup_pis.yml`)
- Pi3 accessible via SSH
- Edit playbook vars to change default password

**Usage**:

```bash
# Edit password in playbook first!
vim ansible/playbooks/setup_pihole.yml  # Change pihole_webpassword

# Deploy Pi-hole
ansible-playbook -i inventory.yml playbooks/setup_pihole.yml --ask-become-pass
```

**Tags**: `setup`, `dns`, `start`, `firewall`, `verify`

**Access**:

- Web interface: http://192.168.8.12:8080/admin
- Default password: `admin` (change this!)

**Post-setup**:

Configure GL.iNet router to use Pi-hole as DNS:

```bash
# On router via Ansible or manually
uci set dhcp.lan.dhcp_option='6,192.168.8.12'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

---

## Adding New Playbooks

When creating new playbooks, document them here with:

- Target hosts/groups
- Purpose (one-line summary)
- What it configures
- Usage example
- Available tags (if any)
