# Homelab Infrastructure Project

A simplified DevOps learning environment built around two identical laptops, one router, and an optional Raspberry Pi for alerts/redundancy.

---

## Overview

This homelab serves as a practical platform for learning and demonstrating DevOps and platform engineering skills. The project emphasizes:

- **Reproducible infrastructure** - Identical prod and staging environments
- **Configuration as code** - Everything automated via Ansible
- **Safe testing** - Deploy to staging, validate, then promote to production
- **Reliable alerting** - Get notified if something breaks
- **Production-like workflows** - Real-world deployment practices on a small scale

---

## Current Infrastructure

### Hardware

**Core devices:**

- **homelab-laptop** (192.168.8.10) - **Production**: main services, always on
- **homelab-staging** (192.168.8.11) - **Staging**: identical setup, brought up as needed for testing/upgrades
- **monitoring-pi** (192.168.8.20, optional) - dedicated monitoring and alerts
- **GL.iNet SF1200 router** (192.168.8.1) - DHCP, WireGuard VPN, DuckDNS, firewall
- **Netgear LM1200 modem** - internet uplink

### Software Stack

**Base:**

- OS: Ubuntu Server 24.04 LTS (both laptops)
- Automation: Ansible for everything
- Containers: Docker + Docker Compose
- Reverse proxy: Nginx

**Services:**

- **Monitoring**: Prometheus, Grafana, Loki
- **Availability**: Uptime Kuma (with Discord/email/webhook alerts)
- **CI/CD**: Jenkins
- **Container management**: Portainer
- **Dashboard**: Homer (service links)
- **DNS**: Pi-hole (ad blocker)
- **Applications**: Portfolio website

**Optional (Pi):**

- Lightweight Prometheus/Grafana mirror
- Independent alerting agent
- Network monitoring

---

## Architecture Decisions

### Two-Laptop Prod/Staging Model

Identical x86 laptops enable true environment separation:

- **Production (192.168.8.10)**: Main deployment, daily traffic
- **Staging (192.168.8.11)**: Exact copy, brought up on-demand for testing

**Advantages:**

- Test service upgrades safely on staging before touching production
- Develop features in isolation
- Identical architecture = no surprises when promoting staging→prod
- Staging laptop can be powered off when not in use (save electricity)

### Optional Pi for Alerts

The Pi runs independent monitoring:

- Checks if the main laptop is reachable
- Can send notifications even if production is down
- Lightweight, low-power, always-on if enabled
- Not a hard requirement—production runs fine without it

### Network Design

Single flat LAN (192.168.8.0/24):

- Static DHCP reservations for all devices
- WireGuard VPN for remote access
- DuckDNS for dynamic public IP
- Pi-hole for network-wide ad blocking
- Static IP assignment and documentation

### Phase 2: Automation ✅ Complete

- Ansible control node setup
- SSH key-based authentication across all hosts
- Base playbooks: system updates, user management, Docker installation
- Router automation with OpenWrt/UCI

### Phase 3: Core Services 🔄 In Progress

- WireGuard VPN server on router
- Laptop becomes the primary runtime for services and automation
- Monitoring and notifications move to the laptop first, with optional Pi redundancy

---

## Quick Start

### Deploy Prod From Scratch (Fresh Ubuntu 24.04 LTS)

```bash
# 1. Copy SSH key to laptop
ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@192.168.8.10

# 2. Deploy base infrastructure
cd ansible
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l prod

# 3. Deploy all services
ansible-playbook -i inventory.yml playbooks/setup_environment.yml -K -l prod

# 4. Access services
#  - Homer dashboard: http://192.168.8.10:8081
#  - Grafana: http://192.168.8.10:3001 (admin/admin)
#  - Jenkins: http://192.168.8.10:9080/jenkins
#  - Portainer: http://192.168.8.10:9000
#  - Portfolio: http://192.168.8.10
```

### Bring Up Staging (For Testing)

```bash
# 1. Copy SSH key to staging laptop
ssh-copy-id -i ~/.ssh/id_ed25519.pub dcsicsak@192.168.8.11

# 2. Deploy base + services
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l staging
ansible-playbook -i inventory.yml playbooks/setup_environment.yml -K -l staging

# 3. Test at http://192.168.8.11:8081 (Homer)
```

### Router Setup

```bash
# Copy DHCP var file and update MAC addresses
cp ansible/vars/router_dhcp.yml.example ansible/vars/router_dhcp.yml
vim ansible/vars/router_dhcp.yml  # Add real MACs

# Deploy router
export DUCKDNS_TOKEN="your-token"  # If using DuckDNS
ansible-playbook -i inventory.yml playbooks/setup_glinet_complete.yml
```

---

## Project Phases

### Phase 1: Foundation ✅ Complete

- Hardware assembled and networked
- Operating systems installed (Ubuntu 24.04 LTS)
- Router configured (DHCP, static reservations)

### Phase 2: Automation & Deployment ✅ Complete

- Ansible playbooks for all infrastructure
- Docker Compose services deployed
- Prod services running and accessible

### Phase 3: Staging Environment ✅ Complete

- Two-laptop prod/staging topology
- Identical deployments for safe testing
- Staging laptop deployable on-demand

### Phase 4: Observability & Alerting 🔄 In Progress

- Prometheus metrics ✅
- Grafana dashboards ✅
- Loki log aggregation ✅
- Uptime Kuma availability checks ✅
- Discord/email alerting setup needed

### Phase 5: CI/CD Maturity

- Jenkins pipeline examples
- Automated testing and deployment
- GitHub integration

### Phase 6: Advanced Experiments

- Local Kubernetes (k3d) for pod learning
- Terraform for future cloud deployments
- Network segmentation via VLANs

---

## Repository Structure

```
homelab/
├── README.md                       # This file
├── .env.example                    # Environment variables template
├── ansible/
│   ├── ansible.cfg                 # Ansible configuration
│   ├── inventory.yml               # Inventory (prod, staging, router, monitoring)
│   ├── playbooks/
│   │   ├── README.md               # Detailed playbook docs
│   │   ├── setup_servers.yml       # Base x86 server setup
│   │   ├── setup_environment.yml   # Master orchestration playbook
│   │   ├── setup_monitoring_stack.yml
│   │   ├── setup_uptime_kuma.yml
│   │   ├── setup_jenkins.yml
│   │   ├── setup_portainer.yml
│   │   ├── setup_pihole.yml
│   │   ├── setup_homer.yml
│   │   ├── setup_portfolio_site.yml
│   │   ├── setup_glinet_complete.yml
│   │   ├── setup_glinet_openwrt.yml
│   │   ├── setup_glinet_ddns.yml
│   │   ├── setup_glinet_wireguard.yml
│   │   ├── bootstrap_glinet.yml
│   │   └── configure_wireguard.yml
│   └── vars/
│       └── router_dhcp.yml.example
├── docs/
│   ├── ansible.md                  # Ansible setup & SSH keys
│   ├── network_setup.md            # Router & network config
│   ├── network_inventory.md        # Device list and IPs
│   ├── software_stack.md           # Service descriptions
│   ├── services_dashboard.md       # Homer dashboard config
│   ├── prod_staging.md             # Prod/staging workflows
│   ├── alerting.md                 # Alerting strategy
│   ├── hardware.md                 # Hardware inventory
│   └── populate_monitoring_services.md
└── docker/  # Docker Compose files (embedded in playbooks currently)
```

---

## Key Features

**Prod/Staging Separation:**

- Identical x86 laptops running identical services
- Test upgrades and changes on staging before production
- Staging laptop powers off when not needed (cost savings)

**Automated Deployment:**

- Single Ansible command deploys entire environment
- Idempotent playbooks (safe to run repeatedly)
- Tagged tasks for selective deployment

**Reliable Alerting:**

- Uptime Kuma monitors service availability
- Grafana tracks resource metrics
- Discord/email notifications for issues
- Optional Pi can alert independently if main laptop is down

**Production-Ready:**

- Nginx reverse proxy
- UFW firewall with proper rules
- Docker Compose for multi-service orchestration
- Persistent storage for all services

---

## Typical Workflows

### Test a Service Upgrade

```bash
# 1. Update image version in playbook
vim playbooks/setup_jenkins.yml

# 2. Deploy to staging
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -l staging

# 3. Test at http://192.168.8.11:9080/jenkins

# 4. If successful, deploy to prod
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml -K -l prod
```

### Develop a Feature on Portfolio

```bash
# 1. Clone your portfolio repo locally
git clone https://github.com/yourusername/portfolio.git

# 2. Push to dev branch
git checkout -b feature/my-changes

# 3. Update playbook to use your branch
vim playbooks/setup_portfolio_site.yml

# 4. Deploy to staging
ansible-playbook -i inventory.yml playbooks/setup_portfolio_site.yml -K -l staging

# 5. Test at http://192.168.8.11
# 6. Merge to main and deploy to prod
```

### Check What's Running

```bash
# Prod services
ssh dcsicsak@192.168.8.10 'docker ps'

# Staging services
ssh dcsicsak@192.168.8.11 'docker ps'

# Prod logs
ssh dcsicsak@192.168.8.10 'docker logs jenkins'
```

---

## Documentation

See the `docs/` directory for detailed information:

- **[ansible.md](docs/ansible.md)** - How to set up Ansible, SSH keys, inventory
- **[network_setup.md](docs/network_setup.md)** - Router configuration, DHCP, WireGuard, DuckDNS
- **[prod_staging.md](docs/prod_staging.md)** - Prod/staging deployment workflows
- **[software_stack.md](docs/software_stack.md)** - Service descriptions and ports
- **[alerting.md](docs/alerting.md)** - Monitoring and notification setup
- **[ansible/playbooks/README.md](ansible/playbooks/README.md)** - Detailed playbook documentation

---

## Why This Topology?

**Two x86 laptops instead of one:**

- True environment separation (prod vs. staging)
- Can safely test changes before touching production
- Staging is on-demand (power off to save electricity)
- No container/VM overhead for isolation—full OS separation

**Optional Pi instead of required cluster:**

- Adds independent alerting without complexity
- Can monitor the main laptop even when it's rebooting
- Still useful for learning network monitoring
- Entirely optional—not needed for core functionality

**Single flat network instead of complex segmentation:**

- Keeps focus on infrastructure automation and deployment
- VLAN segmentation can be added later if needed
- Reduces router configuration complexity
- Suitable for home learning environment

---

## Current Status

**✅ Completed:**

- Router bring-up (DHCP, VPN, DDNS)
- Prod laptop base setup and all services
- Monitoring stack (Prometheus, Grafana, Loki)
- Availability monitoring (Uptime Kuma)
- Jenkins, Portainer, Pi-hole, Homer, Portfolio deployed
- Ansible automation for entire stack
- Documentation

**🔄 In Progress:**

- Fine-tuning alerts and notifications
- Pi integration (optional monitoring)

**📋 Future:**

- Local Kubernetes (k3d) experiments
- Terraform for cloud infrastructure
- Additional Grafana dashboards

---

## Contact & License

This is a personal DevOps learning project. Documentation is available under the MIT License.
