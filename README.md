# Homelab Infrastructure Project

A production-like DevOps learning environment built on real hardware, focusing on automation, containerization, and infrastructure as code.

---

## Overview

This homelab serves as a practical platform for learning and demonstrating DevOps and platform engineering skills. The project emphasizes reproducible infrastructure, configuration as code, and production-ready workflows on self-hosted hardware.

**Key focus areas:**

- Configuration management and automation (Ansible)
- Container orchestration (Docker, Kubernetes)
- CI/CD pipeline development (Jenkins)
- Infrastructure as code (Terraform)
- Monitoring and observability (Prometheus, Grafana)
- Network services (VPN, reverse proxy, DNS filtering)

---

## Current Infrastructure

### Hardware

**Compute (x86 Architecture):**

- **homelab-main** (192.168.8.10) - ThinkPad T440 - **OFFLINE** pending OS reinstall, future production server
- **homelab-staging** (192.168.8.11) - Asus X550C - **ACTIVE** temporarily handling production traffic
- **homelab-mgmt** (192.168.8.12) - MiniPC (Celeron, 8GB RAM) - **PLANNED** management/CI-CD node

**Compute (ARM Architecture):**

- 2x Raspberry Pi 4 (4GB RAM) with M.2 SSD storage via Geekworm X862 expansion boards - **ONLINE** running observability services
- 1x Raspberry Pi 3B+ - **DECOMMISSIONED** (unreliable, services moved to homelab-mgmt)

**Networking:**

- Netgear GS308EP managed PoE+ switch
- GL.iNet SF1200 router (OpenWrt-based, VLAN-capable)
- Evodata PoE hats powering all Raspberry Pis

**Physical Infrastructure:**

- Custom rack built from 2020/2040 aluminum extrusions
- Planned 3D-printed cable management and mounting solutions

### Software Stack

**Core Technologies:**

- **OS:** Ubuntu Server 24.04 LTS (x86 hosts), Raspberry Pi OS 64-bit (ARM nodes)
- **Automation:** Ansible for configuration management and provisioning
- **Containers:** Docker and Docker Compose
- **CI/CD:** Jenkins (planned for management node)
- **Reverse Proxy:** Nginx (planned)
- **Monitoring:** Prometheus + Grafana (running on Pis)
- **VPN:** WireGuard server on router
- **DNS/Ad-Blocking:** Pi-hole (running on Pis)
- **Service Dashboard:** Homer (running on Pis)

**Infrastructure as Code:**

- Ansible playbooks for all system configuration
- Dockerfiles and Compose manifests for service deployment
- Terraform for future cloud and hybrid deployments

---

## Architecture Decisions

### Three-Tier Server Design

The x86 infrastructure follows a three-tier separation of concerns:

**Management Node (homelab-mgmt @ .12):**

- Dedicated to DevOps tooling and infrastructure management
- **Jenkins** for CI/CD pipelines with Docker-in-Docker
- **Portainer** for container orchestration UI
- **Docker Registry** for private image storage
- Isolated from production workloads to prevent build jobs from impacting services

**Staging Node (homelab-staging @ .11):**

- Test deployment target for CI/CD pipelines
- Mirror of production environment for pre-release testing
- Currently serving production traffic while main is offline
- Will return to staging role when production is restored

**Production Node (homelab-main @ .10):**

- Currently offline pending OS reinstall
- Future home for stable production workloads
- Resource-intensive applications (databases, app servers)
- Public-facing services with reverse proxy

**Current Status:**

- **Pis:** Operational - Running Grafana, Prometheus, Homer, Pi-hole
- **Staging:** Active - Handling production traffic temporarily
- **Management:** Planned - MiniPC awaiting setup
- **Main:** Offline - Pending OS reinstall

### Kubernetes Strategy

The project will introduce Kubernetes through k3s (lightweight k8s) on the Raspberry Pi 4 cluster:

- Pi4s provide persistent SSD storage suitable for etcd and workload data
- Separation of concerns: k3s for orchestration learning, x86 hosts for resource-intensive services
- Progression path: standalone k3s cluster → optional mixed-architecture cluster → cloud migration

**Rationale:** This approach balances learning objectives with operational stability. Resource-heavy tasks (Jenkins builds, databases) remain on more capable x86 hardware while Kubernetes workloads run on dedicated ARM nodes.

### Network Design

**Current:** Flat LAN (192.168.8.0/24) with static IP assignments

**Future:** VLAN segmentation is a possibility but not an immediate priority. Focus remains on automation, service reliability, and monitoring before introducing network complexity.

---

## Project Phases

### Phase 1: Foundation ✅ Complete

- Hardware assembly and network connectivity
- Operating system installation and baseline configuration
- Static IP assignment and documentation

### Phase 2: Automation ✅ Complete

- Ansible control node setup
- SSH key-based authentication across all hosts
- Base playbooks: system updates, user management, Docker installation
- Router automation with OpenWrt/UCI

### Phase 3: Core Services 🔄 In Progress

- WireGuard VPN server deployed on router
- Pi-hole DNS filtering operational on Pis
- Grafana/Prometheus monitoring stack running
- Jenkins deployment on management node (next step)
- Nginx reverse proxy with SSL (planned)
- GitHub webhook integration (planned)

### Phase 4: Observability 🔄 Partial

- Prometheus metrics collection
- Grafana dashboards
- Node exporters on all hosts
- Backup and recovery procedures

### Phase 5: Orchestration

- k3s deployment on Raspberry Pi cluster
- Kubernetes workload migration
- Terraform for infrastructure provisioning
- GitOps workflows

### Phase 6: Cloud Integration

- Hybrid cloud architecture using Terraform
- Workload portability between on-prem and cloud
- Comparative performance and cost analysis

---

## Repository Structure

```
homelab/
├── README.md                    # This file
├── hardware.md                  # Complete hardware inventory
├── software_stack.md            # Software tools and deployment paths
├── docs/                        # Additional documentation
│   └── devlog/                  # Session notes and progress logs
├── ansible/                     # Configuration management playbooks (planned)
├── docker/                      # Container definitions and compose files (planned)
└── terraform/                   # Infrastructure as code (planned)
```

---

## Key Learning Outcomes

This project demonstrates practical experience with:

**Infrastructure & Automation:**

- Bare-metal server provisioning and lifecycle management
- Configuration as code using Ansible
- Idempotent, reproducible infrastructure patterns

**Containerization & Orchestration:**

- Docker multi-stage builds and optimization
- Docker Compose for multi-service applications
- Kubernetes cluster administration (k3s)
- Multi-architecture container deployments (ARM/x86)

**CI/CD & DevOps Practices:**

- Jenkins pipeline development
- Automated build, test, and deployment workflows
- Git-based infrastructure workflows
- Integration between CI and container orchestration

**Networking & Security:**

- Reverse proxy configuration and TLS management
- VPN setup for secure remote access
- Network segmentation planning (future)
- Firewall rule management

**Monitoring & Operations:**

- Metrics collection and visualization
- Service health monitoring
- Backup and disaster recovery planning

---

## Why a Homelab?

Cloud platforms are excellent for production workloads, but a physical homelab offers unique learning advantages:

1. **Full control:** Complete access to the infrastructure stack, from hardware to application layer
2. **Real constraints:** Working with limited resources teaches optimization and efficiency
3. **Failure learning:** Safe environment to break things, troubleshoot, and rebuild
4. **Cost-effective:** Using mostly hardware thats already laying around or cheap vs. ongoing cloud costs for experimentation
5. **Hybrid skills:** Experience managing both on-premises and cloud infrastructure

The ultimate goal is cloud competency, with the homelab serving as a proving ground for infrastructure patterns that will be replicated in cloud environments.

---

## Current Status

**Completed:**

- Hardware assembly and rack construction
- Network configuration with static IPs
- Operating system installation on all nodes
- Documentation structure and initial technical specifications

**In Progress:**

- Ansible playbook development
- SSH key distribution and access control
- Jenkins installation planning

**Next Steps:**

- Execute base Ansible playbooks across all hosts
- Deploy Jenkins on primary server
- Create first CI/CD pipeline for containerized application
- Implement WireGuard VPN on Raspberry Pi 4

---

## Documentation

All infrastructure decisions, configurations, and procedures are documented in Markdown and version-controlled in this repository. Each significant change includes a devlog entry with context, implementation details, and lessons learned.

---

## Contact

This is a personal learning project demonstrating DevOps and platform engineering capabilities. For questions or collaboration opportunities, please reach out via GitHub.

---

## License

Documentation and configuration code in this repository are available under the MIT License. See LICENSE file for details.
