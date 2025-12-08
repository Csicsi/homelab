# Homelab Services Dashboard

## Quick Access URLs

| Service             | URL                              | Credentials      | Status         |
| ------------------- | -------------------------------- | ---------------- | -------------- |
| **Homer Dashboard** | **http://192.168.8.12:8081**     | **N/A**          | **✅ Running** |
| Jenkins             | http://192.168.8.12:9080/jenkins | admin/configured | ✅ Running     |
| Portfolio Site      | http://192.168.8.10 (HTTP/HTTPS) | N/A              | ✅ Running     |
| Pi-hole Admin       | http://192.168.8.12:8080/admin   | admin/admin      | ✅ Running     |
| Portainer           | https://192.168.8.12:9443        | admin/configured | ✅ Running     |
| Prometheus          | http://192.168.8.21:30090        | N/A              | ✅ Running     |
| Grafana             | http://192.168.8.21:30030        | admin/admin      | ✅ Running     |
| GL.iNet Router      | http://192.168.8.1               | root/configured  | ✅ Running     |

---

## Service Breakdown by Host

### homelab-main (192.168.8.10) - x86 ThinkPad T440

**Docker Containers:**

- **Jenkins** (jenkins/jenkins:lts) - ⚠️ TEMPORARY
  - Port: 9080 (HTTP)
  - Purpose: CI/CD automation, build pipelines
  - Status: Running (will be removed)
- **Portfolio Site** (portfolio-portfolio)
  - Ports: 80 (HTTP), 443 (HTTPS)
  - Purpose: Personal portfolio/website
  - Status: Running
- **Minishell API** (portfolio-minishell-api)
  - Port: 3000 (internal)
  - Purpose: API backend for portfolio
  - Status: Running

**System Services:**

- Docker Engine
- Node Exporter (port 9100) - metrics for Prometheus
- SSH (port 22)

---

### pi4-node1 (192.168.8.20) - k3s Server Node

**k3s Workloads:**

- **Prometheus** (NodePort 30090)

  - Metrics collection and storage
  - Scrapes all node exporters (15s interval)
  - 7-day retention

- **Grafana** (NodePort 30030)
  - Visualization dashboards
  - Pre-configured data sources: Prometheus, Loki
- **Loki** (internal)
  - Log aggregation
  - Accessible via Grafana

**System Services:**

- k3s server (control plane)
- Node Exporter (port 9100)
- SSH (port 22)

---

### pi4-node2 (192.168.8.21) - k3s Agent Node

**k3s Role:**

- Worker node (agent)
- Available for pod scheduling
- Part of monitoring cluster

**System Services:**

- k3s agent
- Node Exporter (port 9100)
- SSH (port 22)

---

### homelab-mgmt (192.168.8.12) - MiniPC Management Node

**Docker Containers:**

- **Homer Dashboard** (b4bz/homer:latest)

  - Port: 8081 (HTTP)
  - Purpose: Unified dashboard for all homelab services
  - Status: Running
  - Features: Quick links to all services, infrastructure overview

- **Pi-hole** (pihole/pihole:latest)

  - Ports: 53 (DNS TCP/UDP), 8080 (Web UI)
  - Purpose: Network-wide DNS filtering and ad blocking
  - Status: Running

- **Jenkins** (jenkins/jenkins:lts)

  - Ports: 9080 (HTTP), 50000 (agent)
  - Purpose: CI/CD automation, management node builds
  - Status: Running

- **Portainer** (portainer/portainer-ce:latest)

  - Ports: 9000 (HTTP), 9443 (HTTPS)
  - Purpose: Docker container management
  - Status: Running

**System Services:**

- Docker Engine
- Node Exporter (port 9100) - metrics for Prometheus
- SSH (port 22)

---

### homelab-staging (192.168.8.11) - Staging Server

**Docker Containers:**

- No containers currently running

**System Services:**

- Docker Engine
- SSH (port 22)

---

### pi3-utils (192.168.8.22) - Raspberry Pi 3B+ (Deprecated)

**Status:** Services migrated to homelab-mgmt

**System Services:**

- Docker Engine
- SSH (port 22)

---

### glinet-router (192.168.8.1) - GL.iNet SF1200

**Services:**

- **WireGuard VPN Server**
  - Port: 51820 (UDP)
  - Purpose: Remote access to homelab
- **DuckDNS DDNS**
  - Purpose: Dynamic DNS for changing public IP
- **DHCP Server**

  - Static reservations for all hosts
  - Range: 192.168.8.0/24

- **OpenWrt UCI Management**
  - Firewall (fw4)
  - Network configuration

---

## Monitoring Coverage

All hosts report metrics to Prometheus via Node Exporter:

- ✅ homelab-main (192.168.8.10:9100)
- ✅ pi4-node1 (192.168.8.20:9100)
- ✅ pi4-node2 (192.168.8.21:9100)
- ❓ pi3-utils (192.168.8.22:9100) - needs verification
- ❓ glinet-router (192.168.8.1) - router metrics not yet configured

**Web Applications:**

- Homer Dashboard (port 8081) - **START HERE**
- Portfolio site (port 80/443)
- Jenkins (port 9080)
- Pi-hole Admin (port 8080)
- Grafana (port 30030)
- Prometheus (port 30090)
  **Web Applications:**
- Portfolio site (port 80/443)
- Jenkins (port 9080)
- Pi-hole Admin (port 8080)
- Grafana (port 30030)
- Prometheus (port 30090)

**Infrastructure:**

- DNS: Pi-hole on 192.168.8.12:53
- VPN: WireGuard on router (port 51820)
- DHCP: Router (192.168.8.1)
- Metrics: Prometheus scraping node exporters
- Logs: Loki on k3s cluster
- Container Registry: 192.168.8.12:5000

**Orchestration:**

- k3s: 2-node cluster (pi4-node1 + pi4-node2)
  - Access: `export KUBECONFIG=~/.kube/k3s-config && kubectl get pods -A`
  - Running: Prometheus, Grafana, Loki, CoreDNS, Metrics Server
- Docker: homelab-main, homelab-mgmt, homelab-staging
- Docker Compose: Pi-hole, Homer

---

## Next Steps / TODO

- [ ] Configure router DNS to point to Pi-hole (192.168.8.12)
- [ ] Change Pi-hole admin password from default
- [ ] Create Grafana dashboards for homelab metrics
- [ ] Document Jenkins pipeline configurations
- [ ] Set up automated backups for persistent volumes
- [ ] Investigate NotReady k3s node (raspberrypi)
- [ ] Consider decommissioning pi3-utils (services migrated)
- [ ] Configure Portainer to manage k3s cluster
- [ ] Set up kubectl access for all team members

---

## Maintenance Notes

**Last Updated:** 2025-12-08

**Recent Changes:**

- Homer and Pi-hole migrated from pi3-utils to homelab-mgmt
- kubectl configured locally with access to k3s cluster
- Portainer and Docker Registry deployed on homelab-mgmt
- K3s cluster running: Prometheus, Grafana, Loki
- Added kubectl configuration: `export KUBECONFIG=~/.kube/k3s-config`

**Recent Changes:**

- Pi-hole deployed on pi3-utils (docker-compose)
- Fixed systemd-resolved conflict on Raspberry Pi OS
- Replaced community.docker module with shell for reliability

**Known Issues:**

- None currently

**Backup Status:**

- Jenkins config: Not automated
- Grafana dashboards: Not backed up
- Pi-hole config: Persistent volumes in /opt/pihole
