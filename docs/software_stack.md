# Software Stack

## Purpose

This document describes the operating systems, tools, and services for the simplified homelab. The stack keeps the hardware minimal while preserving clean stage separation through containers, VMs, and local Kubernetes.

---

## Operating Systems

| Host           | OS              | Version    | Role                                                       |
| -------------- | --------------- | ---------- | ---------------------------------------------------------- |
| Homelab laptop | Ubuntu Server   | 24.04 LTS  | Primary host — Ansible control, Docker host, local k8s lab |
| Monitoring Pi  | Raspberry Pi OS | 64-bit     | Optional redundancy node for monitoring, alerting, or DNS  |
| Workstation    | Ubuntu / Debian | Latest LTS | Development and testing                                    |

Notes:

- Ubuntu Server 24.04 LTS chosen for long-term support, Debian compatibility, and wide community adoption
- Raspberry Pi OS is only needed if the optional Pi stays in service
- All systems use 64-bit where possible

---

## Core Tooling

### Automation and Provisioning

- **Ansible**: Centralized configuration management from the homelab laptop
  - Playbooks for router, laptop, and optional monitoring Pi
  - Inventory organized by logical role instead of many physical machines

### Containers and Orchestration

- **Docker**: default runtime for services on the laptop
- **Docker Compose**: primary way to separate production and staging stacks on one host
- **VMs**: optional isolation for risky experiments, alternative operating systems, or firewall labs
- **k3d** or **kind**: lightweight local Kubernetes for pod-based learning without extra devices
  - Good for testing Deployments, Services, Ingress, Helm, and GitOps locally
  - Avoids maintaining a dedicated multi-node Pi cluster

### CI/CD

- **Jenkins**: optional build automation if local CI remains useful
  - Can build Docker images and deploy to Compose or a local k8s lab
  - Should stay only if it supports active workflows

### Networking and Security

- **Nginx** or **Caddy**: Reverse proxy and TLS termination on the laptop
  - HTTP/HTTPS routing for containerized services
  - Let's Encrypt integration for SSL certificates
  - Load balancing for multi-instance deployments
- **WireGuard**: VPN server on GL.iNet router for remote access
- **Pi-hole**: optional DNS filtering via Docker on the laptop or optional Pi
- **Firewall**: UFW on Ubuntu Server, UFW on Raspberry Pis

### Monitoring and Observability

- **Prometheus**: Metrics collection on the laptop, optionally duplicated on the Pi
- **Grafana**: Dashboards and alerting on the laptop
- **Uptime Kuma**: Service and endpoint monitoring with direct notifications
- **Node Exporter**: Host metrics for the laptop and optional Pi
- **Loki**: Optional if logs are useful enough to justify the extra moving part

### Notifications and Alerting

- **Primary recommendation**: Uptime Kuma for service checks and Grafana Alerting for system metrics
- **Notification targets**: ntfy, email, Telegram, or Discord
- **Redundancy option**: run a small checker on the Pi so it can alert when the laptop is down

### Infrastructure as Code (planned Phase 5)

- **Terraform**: Reproducible infrastructure provisioning
  - Start with local resources (libvirt VMs, network config)
  - Extend to cloud providers post-k8s (AWS, GCP, or DigitalOcean)

---

## Service Deployment Path

1. **Docker Compose on the laptop** (Primary path)

- Run daily services in dedicated Compose projects such as `prod`, `staging`, and `ops`
- Keep ports, volumes, and environment files separate by stage

2. **Monitoring on the laptop** (Primary path)

- Run Prometheus, Grafana, and Uptime Kuma locally
- Send alerts to a notification channel instead of relying on dashboards alone

3. **Monitoring on the Pi** (Optional redundancy)

- Run a second notifier or a minimal uptime checker
- Keep one independent path that can alert if the laptop fails

4. **Local Kubernetes lab** (Optional)

- Use `k3d` or `kind` on the laptop for pods, Helm charts, and ingress experiments

5. **Cloud migration** (Future)

- Reuse container and IaC patterns for cloud deployments later

---

## Network Architecture (current)

- Flat LAN: `192.168.8.0/24`
- GL.iNet router handles DHCP, routing, and WireGuard
- Static IP assignments only for the laptop, optional Pi, and infrastructure gear

VLANs are a future possibility but not currently planned. If/when a VLAN-capable router is introduced, segmentation will be considered.

---

## Development Workflow

1. Write playbook or Dockerfile on workstation
2. Commit to GitHub
3. Ansible provisions the router, laptop, and optional Pi
4. Deploy to Docker Compose, a VM, or a local `k3d`/`kind` cluster
5. Monitor with Prometheus/Grafana and alert with Uptime Kuma or Grafana Alerting
6. Document in `docs/` with devlog entry

---

## Summary

The software stack prioritizes open-source, widely-adopted tools with less hardware overhead. The progression is: automation (Ansible) → containers (Docker Compose) → optional isolation (VMs) → local orchestration (`k3d`/`kind`) → IaC and cloud when needed.
