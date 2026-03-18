# Populating Prometheus, Grafana, and Notifications

## Overview

This guide assumes a simplified monitoring layout:

- Prometheus, Grafana, and Uptime Kuma run on the laptop
- The optional Pi acts as an independent watcher or secondary notifier
- Pi-hole is optional and should only be configured if you actually keep it in the stack

---

## 1. Prometheus Configuration

### Current Target State

Prometheus should scrape at least:

- `homelab-laptop` on `192.168.8.10:9100`
- `monitoring-pi` on `192.168.8.20:9100` when enabled

### What to Check

1. Open Prometheus:

   ```
   http://192.168.8.10:30090
   ```

2. Verify basic queries:
   - `up`
   - `node_cpu_seconds_total`
   - `node_memory_MemAvailable_bytes`
   - `node_filesystem_avail_bytes`

3. Check targets under **Status → Targets** and confirm the laptop is always present.

### Recommended Scrape Targets

- Node Exporter on the laptop
- Node Exporter on the Pi
- Optional application metrics if a service exposes them

### Retention and Performance

- Keep a short retention window at first, such as 7 days
- Increase only if the data is genuinely useful

---

## 2. Grafana Setup

### Initial Login

- URL: `http://192.168.8.10:30030`
- Change the default admin password immediately

### Essential Dashboards

Start with only a few dashboards:

1. **Host Overview**
   - CPU usage
   - Memory usage
   - Disk usage
   - Uptime

2. **Per-Host Detail**
   - Filter by host label
   - Show CPU, memory, disk, and network trends

3. **Service Health**
   - Combine Prometheus metrics with links to Uptime Kuma

### Alert Rules to Add First

- CPU above 80% for 5 minutes
- Memory above 85%
- Disk usage above 90%
- `up == 0` for the laptop or Pi exporters

### Notification Channels

Connect Grafana to one of:

- ntfy
- Email
- Telegram
- Discord webhook

---

## 3. Uptime Kuma Setup

### Why Use It

Uptime Kuma is the easiest way to get immediate service-down alerts without building everything around Prometheus rules.

### Suggested Monitors

- Ping: `192.168.8.1` for the router
- Ping: `192.168.8.10` for the laptop
- HTTP: Homer on `http://192.168.8.10:8081`
- HTTP: Grafana on `http://192.168.8.10:30030`
- HTTP: Prometheus on `http://192.168.8.10:30090`
- Optional HTTP or TCP checks for any app stack you care about

### Redundancy Option

If the Pi stays online, run a second Uptime Kuma instance there and have it check the laptop. That way the monitor is still alive if the main host is down.

---

## 4. Pi-hole Configuration (Optional)

Only do this if Pi-hole remains part of your final design.

### Suggested DNS Records

```text
homelab-laptop.local  → 192.168.8.10
monitoring-pi.local   → 192.168.8.20
grafana.local         → 192.168.8.10
prometheus.local      → 192.168.8.10
uptime.local          → 192.168.8.10
```

### Router DNS Recommendation

- If Pi-hole runs on the laptop, point router DNS to `192.168.8.10`
- If Pi-hole runs on the Pi, point router DNS to `192.168.8.20`
- Keep a fallback public resolver only if needed

---

## 5. Minimal Useful Setup

If you want the smallest setup that still notifies you:

1. Run Node Exporter on the laptop
2. Run Prometheus and Grafana on the laptop
3. Run Uptime Kuma on the laptop
4. Connect Uptime Kuma to ntfy, Telegram, or Discord
5. Optionally run a second Uptime Kuma on the Pi

That gets you from dashboards-only to real notifications with very little overhead.

2. **Enable query logging:**
   - Shows real-time DNS queries
   - Useful for debugging and monitoring

3. **View analytics:**
   - Dashboard tab shows:
     - Queries blocked %
     - Top blocked domains
     - Top permitted domains
     - Top clients

### Step 5: Whitelisting and Blacklisting

1. **Whitelist domains** (Settings → Whitelist):
   - Domains that should never be blocked
   - Example: `github.com` if GitHub CDN is mistakenly blocked

2. **Blacklist domains** (Settings → Blacklist):
   - Domains to always block
   - Example: specific tracking domains

3. **Regex filters** (Settings → Regex Filter):
   - Advanced pattern matching for blocks
   - Example: `^ads.*\.` blocks all subdomains under `ads.`

### Step 6: Group Management (Optional)

1. **Adlist Management → Group**
2. Create custom groups to organize blocklists
3. Assign different groups to different clients if needed

### Step 7: Monitor Pi-hole from Grafana (Optional)

1. **Export Pi-hole metrics to Prometheus:**
   - Install Pi-hole exporter (requires additional setup)
   - Or query Pi-hole API directly in Grafana

2. **Add Pi-hole dashboard:**
   - Search Grafana dashboards for "Pi-hole"
   - Import community dashboard (ID: 13048)

---

## 4. Integration: Connecting Everything

### A. Add Pi-hole Data to Prometheus

1. **Install Pi-hole exporter on pi3-utils:**

   ```bash
   docker run -d \
     -p 9311:9311 \
     --name pihole-exporter \
     -e PIHOLE_HOSTNAME=192.168.8.12 \
     -e PIHOLE_PORT=8080 \
     -e PIHOLE_PROTOCOL=http \
     pihole/pihole-exporter
   ```

2. **Add to Prometheus config:**
   - Edit ConfigMap in `setup_monitoring_stack.yml`
   - Add scrape job:
     ```yaml
     - job_name: "pihole"
       static_configs:
         - targets: ["192.168.8.12:9311"]
     ```

3. **Recreate Prometheus pod:**
   ```bash
   kubectl rollout restart deployment/prometheus -n monitoring
   ```

### B. Create Unified Dashboard

1. **In Grafana, create new dashboard:**
2. **Add panels from all three sources:**
   - Prometheus: System metrics
   - Pi-hole exporter: DNS blocking stats
   - Logs from Loki (if available)

3. **Example combined panel:**
   ```
   Title: "Network Health"
   Metrics:
   - CPU load
   - Memory usage
   - DNS queries blocked %
   - Active connections
   ```

---

## 5. Troubleshooting

### Prometheus Issues

| Problem               | Solution                                                    |
| --------------------- | ----------------------------------------------------------- |
| No metrics showing    | Check "Status → Targets", verify all targets are "UP"       |
| Targets DOWN          | SSH to host and check `sudo systemctl status node_exporter` |
| Query returns no data | Verify metric names with `node_` prefix exist               |

### Grafana Issues

| Problem                     | Solution                                                  |
| --------------------------- | --------------------------------------------------------- |
| Can't connect to Prometheus | Check Settings → Data Sources, click Test                 |
| Panels show "No data"       | Verify Prometheus is scraping (go to Prometheus UI)       |
| Dashboards very slow        | Reduce query time range, add `rate()` functions to graphs |

### Pi-hole Issues

| Problem              | Solution                                                           |
| -------------------- | ------------------------------------------------------------------ |
| Ads still showing    | Check if router DNS is pointing to Pi-hole (192.168.8.12)          |
| High false positives | Disable some blocklists or add to whitelist                        |
| DNS not resolving    | Check if systemd-resolved is stopped (playbook should handle this) |
| Web UI slow          | Reduce query logging load or enable compression in settings        |

---

## 6. Next Steps

1. **Immediate:**
   - ✅ Verify metrics in Prometheus
   - ✅ Create basic dashboards in Grafana
   - ✅ Configure Pi-hole blocklists
   - ✅ Change default passwords

2. **Short-term (this week):**
   - Add Pi-hole metrics to Prometheus
   - Create unified monitoring dashboard
   - Set up Grafana alerts
   - Configure router to use Pi-hole DNS

3. **Long-term (this month):**
   - Add application metrics (Jenkins, Docker)
   - Set up log aggregation with Loki
   - Create custom dashboards per service
   - Integrate with alerting system (Slack, email)

---

## Useful Commands

```bash
# Check node exporter status
systemctl status node_exporter

# View node exporter metrics
curl http://192.168.8.20:9100/metrics | grep node_memory

# Check Prometheus targets
curl http://192.168.8.20:30090/api/v1/targets

# Check Grafana datasources
curl http://192.168.8.20:30030/api/datasources

# Query Prometheus directly
curl 'http://192.168.8.20:30090/api/v1/query?query=up'

# Access Pi-hole API
curl http://192.168.8.12:8080/admin/api.php?summaryRaw

# SSH to monitoring node
ssh pi@192.168.8.20
```

---

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Dashboard Library](https://grafana.com/grafana/dashboards)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Node Exporter Metrics](https://github.com/prometheus/node_exporter#enabled-by-default)
- [PromQL Query Language](https://prometheus.io/docs/prometheus/latest/querying/basics/)
