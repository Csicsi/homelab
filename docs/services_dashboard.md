# Homelab Services Dashboard

## Quick Access URLs

| Service             | URL                              | Credentials      | Status            |
| ------------------- | -------------------------------- | ---------------- | ----------------- |
| **Homer Dashboard** | **http://192.168.8.10:8081**     | **N/A**          | **Primary entry** |
| Jenkins             | http://192.168.8.10:9080/jenkins | admin/configured | Optional          |
| Portfolio Site      | http://192.168.8.10              | N/A              | Optional          |
| Pi-hole Admin       | http://192.168.8.10:8080/admin   | admin/configured | Optional          |
| Portainer           | https://192.168.8.10:9443        | admin/configured | Optional          |
| Prometheus          | http://192.168.8.10:30090        | N/A              | Recommended       |
| Grafana             | http://192.168.8.10:30030        | admin/configured | Recommended       |
| Uptime Kuma         | http://192.168.8.10:3001         | configured       | Recommended       |
| GL.iNet Router      | http://192.168.8.1               | root/configured  | Required          |

---

## Runtime Layout

### homelab-laptop (192.168.8.10)

**Primary roles:**

- Main Docker or Docker Compose host
- Ansible control node
- Optional local Kubernetes lab using `k3d` or `kind`
- Main monitoring and alerting node

**Typical services:**

- Homer
- Portainer
- Jenkins if still useful
- Pi-hole if you want local DNS filtering here
- Prometheus
- Grafana
- Uptime Kuma
- Application stacks such as portfolio or experiments

**Suggested stage separation:**

- `prod` Compose project for stable daily services
- `staging` Compose project or VM for pre-production testing
- `lab` namespace, VM, or local cluster for pod experiments

---

### monitoring-pi (192.168.8.20, optional)

**Primary roles:**

- Independent watcher for laptop and router uptime
- Backup notification path
- Optional secondary DNS or small monitoring components

**Recommended lightweight services:**

- Uptime Kuma
- Node Exporter
- ntfy relay or webhook forwarder if desired

---

### glinet-router (192.168.8.1)

**Services:**

- WireGuard VPN server
- DHCP reservations
- Optional DuckDNS DDNS
- Firewall and LAN management

---

## Monitoring Coverage

**Core targets:**

- ✅ homelab-laptop at `192.168.8.10:9100`
- ✅ monitoring-pi at `192.168.8.20:9100` when enabled
- ⚠️ GL.iNet router availability by ping or HTTP check

**Recommended web apps:**

- Homer as the landing page
- Grafana for dashboards and alert rules
- Prometheus for metrics
- Uptime Kuma for endpoint monitoring and notifications

---

## Next Steps

- [ ] Finalize whether Pi-hole lives on the laptop, the Pi, or not at all
- [ ] Deploy Uptime Kuma and connect one notification channel
- [ ] Add Grafana alerts for CPU, memory, and disk thresholds
- [ ] Decide whether staging should be a Compose project or a VM
- [ ] Keep the Pi online only if it adds real redundancy value

---

## Maintenance Notes

**Last Updated:** 2026-03-17

**Current direction:**

- Single-host-first homelab on the laptop
- Optional Pi for independent alerting and small redundancy tasks
- Router remains the edge device for WireGuard and DHCP
