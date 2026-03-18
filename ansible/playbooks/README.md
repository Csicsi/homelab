# Ansible Playbooks

## Overview

Playbooks deploy the homelab on a **two-laptop + optional monitoring Pi** topology:

- **Production (192.168.8.10)**: x86 laptop, main services
- **Staging (192.168.8.11)**: x86 laptop, identical setup for safe testing/upgrades
- **Router (192.168.8.1)**: GL.iNet SF1200 with OpenWrt
- **Monitoring Pi (192.168.8.20)**: Optional - alerts, redundancy, central metrics

---

## Quick Start

### 1. Fresh Server Setup (Ubuntu 24.04 LTS)

```bash
# Deploy base packages, Docker, firewall, Nginx
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K

# Target specific environment
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l prod
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l staging
```

### 2. Deploy All Services

```bash
# Deploy complete stack to both prod and staging
ansible-playbook -i inventory.yml playbooks/setup_environment.yml -K

# Or deploy individual services
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -l prod
ansible-playbook -i inventory.yml playbooks/setup_monitoring_stack.yml -K -l staging
```

### 3. Router Configuration

```bash
# One-time complete router setup
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml

# Or run individual components
ansible-playbook -i inventory.yml playbooks/setup_glinet_openwrt.yml
ansible-playbook -i inventory.yml playbooks/setup_glinet_ddns.yml
ansible-playbook -i inventory.yml playbooks/setup_glinet_wireguard.yml
```

---

## Available Playbooks

### Server Infrastructure

#### `setup_servers.yml`

Base configuration for x86 servers (prod + staging):

- System packages and Docker
- UFW firewall (SSH, HTTP, HTTPS)
- Nginx reverse proxy
- Laptop lid config (stay running with lid closed)

#### `setup_environment.yml`

Master orchestration playbook - deploys all services in sequence:

- Runs `setup_monitoring_stack.yml`
- Runs `setup_homer.yml`
- Runs `setup_uptime_kuma.yml`
- Runs `setup_jenkins.yml`
- Runs `setup_portainer.yml`
- Runs `setup_pihole.yml`
- Runs `setup_portfolio_site.yml`

### Monitoring & Observability

#### `setup_monitoring_stack.yml`

Deploy Prometheus, Grafana, Loki via Docker Compose:

- Prometheus for metrics collection (port 30090)
- Grafana dashboards + alerting (port 3001, admin/admin)
- Loki for log aggregation (port 3100)
- node_exporter on all hosts (systemd service)

#### `setup_uptime_kuma.yml`

Deploy Uptime Kuma availability monitoring:

- Service status checks (HTTP/TCP/DNS/Ping)
- Status page and notifications (Discord, email, webhook)
- Port 3001

### CI/CD & Management

#### `setup_jenkins.yml`

Deploy Jenkins CI/CD server:

- Docker-in-Docker support
- Ports: 9080 (web), 50000 (agents)
- Persistent storage: `/srv/jenkins`

#### `setup_portainer.yml`

Deploy Portainer container management UI:

- Web-based Docker management
- Ports: 9000 (HTTP), 9443 (HTTPS)
- Persistent storage: `/srv/portainer`

### Network & DNS

#### `setup_pihole.yml`

Deploy Pi-hole DNS server and ad blocker:

- Network-wide ad blocking
- Ports: 53 (DNS), 8080 (web admin)
- Upstream DNS: Cloudflare (configurable)

### Dashboard & Applications

#### `setup_homer.yml`

Deploy Homer service dashboard:

- Unified links to all homelab services
- Port: 8081

#### `setup_portfolio_site.yml`

Deploy Portfolio website from GitHub:

- Clone repository (Csicsi/Portfolio)
- Docker Compose deployment
- HTTP-only LAN mode (no HTTPS redirect)
- Port: 80

### Router Configuration

#### `setup_glinet_complete.yml`

Complete router setup pipeline (recommended):

1. Bootstrap Python on router
2. Configure network/DHCP
3. Setup DuckDNS (optional)
4. Configure WireGuard VPN (optional)

**Prerequisites**:

- SSH RSA key deployed to router
- `ansible/vars/router_dhcp.yml` configured with MAC addresses

**Usage**:

```bash
export DUCKDNS_TOKEN="your-token"
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml
```

#### `bootstrap_glinet.yml`

Install Python3 on router (one-time, before other router playbooks)

#### `setup_glinet_openwrt.yml`

Configure network, DHCP, and static DHCP reservations via UCI

**Prerequisites**:

- Copy vars file: `cp ansible/vars/router_dhcp.yml.example ansible/vars/router_dhcp.yml`
- Update MAC addresses in `router_dhcp.yml`

#### `setup_glinet_ddns.yml`

Configure DuckDNS for dynamic DNS (changing public IP)

```bash
export DUCKDNS_TOKEN="your-token"
ansible-playbook -i inventory.yml playbooks/setup_glinet_ddns.yml
```

#### `setup_glinet_wireguard.yml`

Configure WireGuard VPN server on router:

- Port 51820 UDP
- LAN access via VPN
- Firewall rules

---

## Inventory Groups

| Group           | Hosts                          | Purpose               |
| --------------- | ------------------------------ | --------------------- |
| `prod`          | homelab-laptop (192.168.8.10)  | Production            |
| `staging`       | homelab-staging (192.168.8.11) | Staging/testing       |
| `servers`       | prod + staging                 | Both x86 servers      |
| `services_host` | prod + staging                 | Services deployment   |
| `monitoring`    | monitoring-pi (192.168.8.20)   | Monitoring (optional) |
| `routers`       | glinet-router (192.168.8.1)    | Router                |

---

## Common Workflows

### Deploy Full Prod Stack (Fresh OS)

```bash
# 1. Base server
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l prod

# 2. All services
ansible-playbook -i inventory.yml playbooks/setup_environment.yml -K -l prod
```

### Bring Up Staging Laptop

```bash
# 1. Deploy to staging
ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@192.168.8.11
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l staging
ansible-playbook -i inventory.yml playbooks/setup_environment.yml -K -l staging

# 2. Access Homer: http://192.168.8.11:8081
```

### Test Service Upgrade on Staging

```bash
# 1. Edit playbook with new version
vim playbooks/setup_jenkins.yml

# 2. Deploy to staging first
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -l staging

# 3. Test at http://192.168.8.11:9080/jenkins

# 4. If successful, deploy to prod
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -l prod
```

### Update Single Service on Prod

```bash
ansible-playbook -i inventory.yml playbooks/setup_homer.yml -K -l prod
```

---

## Troubleshooting

### Test Connectivity

```bash
ansible -i inventory.yml prod -m ping
ansible -i inventory.yml staging -m ping
ansible -i inventory.yml glinet-router -m ping
```

### Check Playbook Syntax

```bash
ansible-playbook -i inventory.yml playbooks/setup_servers.yml --syntax-check
```

### Verbose Output

```bash
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -vv
```

### Check Remote Service Status

```bash
ssh dcsicsak@192.168.8.10 'docker ps'
ssh dcsicsak@192.168.8.10 'docker logs jenkins'
```

---

## Environment Variables

### Router Setup

```bash
# Required for DuckDNS
export DUCKDNS_TOKEN="your-token-from-duckdns.org"

# Optional: custom domain
export DUCKDNS_DOMAIN="myname"  # For myname.duckdns.org
```

---

## Related Documentation

- **Prod/Staging Workflows**: See `docs/prod_staging.md`
- **Network Setup**: See `docs/network_setup.md`
- **Ansible Configuration**: See `docs/ansible.md`
- **Service Dashboard**: See `docs/services_dashboard.md`
