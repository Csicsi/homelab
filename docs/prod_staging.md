# Production / Staging Environment Separation

## Overview

The homelab is now split into two environments to support safe testing and development:

- **Production (Laptop @ 192.168.8.10)**: Main instance, always stable
- **Staging (Pi @ 192.168.8.20)**: Test instance for upgrades, feature development, and validations

## Architecture

```
Production                          Staging
┌─ homelab-laptop ─┐             ┌─ monitoring-pi ─┐
├─ Portfolio (80)  │             ├─ Portfolio      │
├─ Jenkins (9080)  │             ├─ Jenkins        │
├─ Portainer (9443)│             ├─ Portainer      │
├─ Pi-hole (8080)  │             ├─ Pi-hole        │
├─ Homer (80)      │  ←────────→ ├─ Homer          │
├─ Grafana (3001)  │             ├─ Grafana        │
├─ Prometheus      │             ├─ Prometheus     │
├─ Loki            │             ├─ Loki           │
└─ Uptime Kuma     │             └─ Uptime Kuma    │
```

## Using Inventory Groups

### Deploy to Production Only

```bash
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l prod
```

### Deploy to Staging Only

```bash
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K -l staging
```

### Deploy to Both

```bash
ansible-playbook -i inventory.yml playbooks/setup_servers.yml -K
```

## Typical Workflows

### 1. Testing a Service Upgrade

**On Staging:**

```bash
# Update the playbook with new image version
# Deploy to staging first
ansible-playbook -i inventory.yml playbooks/setup_monitoring_stack.yml -K -l staging

# Verify it works at http://192.168.8.20:3001 (Grafana example)
# Test functionality, check logs
docker logs grafana  # on the Pi

# Once validated, apply to production
ansible-playbook -i inventory.yml playbooks/setup_monitoring_stack.yml -K -l prod
```

### 2. Developing the Portfolio Site

**Option A: Direct Staging Deployment**

```bash
# Clone portfolio to staging Pi
ansible-playbook -i inventory.yml playbooks/setup_portfolio_site.yml -K -l staging

# Access at http://192.168.8.20:80
# Make changes, test, validate
# When ready, deploy to production
ansible-playbook -i inventory.yml playbooks/setup_portfolio_site.yml -K -l prod
```

**Option B: Local Development → Staging → Production**

```bash
# Work locally on your machine
git clone https://github.com/Csicsi/Portfolio.git

# Push to a dev branch
git push origin feature/my-changes

# Update the playbook to checkout your branch
# Deploy to staging
ansible-playbook -i inventory.yml playbooks/setup_portfolio_site.yml -K -l staging

# Test at http://192.168.8.20
# Once validated, merge to main and deploy to prod
```

### 3. Jenkins Pipeline Testing

**Staging Jenkins:**

```bash
# Test job configurations on staging instance
# Access at http://192.168.8.20:9080/jenkins

# Once pipeline works, replicate to production
# Access production at http://192.168.8.10:9080/jenkins
```

## Important Considerations

### Pi Resource Constraints

The monitoring Pi has limited resources. Consider:

- Running essential containers only (Portfolio, Jenkins core, Grafana)
- Disabling heavy workloads (Prometheus archival, large log retention)
- Using Prometheus on laptop as central metrics store for both envs

### Network & DNS

- Both environments share the same router and DNS
- Homer on each env shows local endpoints
- For external access: use primary router IP (or DuckDNS) + port targeting

### Data Isolation

Services maintain separate data directories:

- **Prod**: `/srv/jenkins`, `/srv/grafana`, etc. on laptop
- **Staging**: `/srv/jenkins`, `/srv/grafana`, etc. on Pi

### Syncing Configurations

When you want staging to match production:

```bash
# Deploy the same playbook to both
ansible-playbook -i inventory.yml playbooks/setup_jenkins.yml
```

Or copy specific configs:

```bash
# Copy Jenkins job definitions from prod to staging
ansible-playbook -i inventory.yml -m copy -a 'src=/srv/jenkins/jobs dest=/srv/jenkins/jobs' monitoring-pi
```

## Monitoring Both Environments

Uptime Kuma and Prometheus on the laptop can monitor both:

- **Laptop endpoints**: `192.168.8.10:*`
- **Pi endpoints**: `192.168.8.20:*`

Configure health checks and alerts to catch issues in either environment.

## Rollback Strategy

If production breaks during an upgrade:

1. Revert the playbook change
2. Redeploy to production: `ansible-playbook -i inventory.yml playbooks/setup_X.yml -K -l prod`
3. Use staging as reference if needed: `docker logs <service> -f` on Pi to compare behavior

## Quick Reference

| Task                           | Command                                                           |
| ------------------------------ | ----------------------------------------------------------------- |
| Deploy all services to prod    | `ansible-playbook -i inventory.yml playbooks/*.yml -K -l prod`    |
| Deploy all services to staging | `ansible-playbook -i inventory.yml playbooks/*.yml -K -l staging` |
| Check prod status              | `ssh dcsicsak@192.168.8.10 'docker ps'`                           |
| Check staging status           | `ssh dcsicsak@192.168.8.20 'docker ps'`                           |
| View prod logs                 | `ssh dcsicsak@192.168.8.10 'docker logs <container>'`             |
| View staging logs              | `ssh dcsicsak@192.168.8.20 'docker logs <container>'`             |
