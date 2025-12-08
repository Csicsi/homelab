# Populating Prometheus, Grafana, and Pi-hole

## Overview

Your monitoring stack is running but needs configuration to be useful. This guide walks through populating each service with real data and dashboards.

---

## 1. Prometheus Configuration

### Current Status

✅ Prometheus is running and scraping node exporters from all hosts (15s interval)

### What to Check

1. **Verify metrics are being collected:**

   ```
   Visit: http://192.168.8.20:30090
   ```

2. **Access the query interface:**

   - Click "Graph" tab
   - In the expression field, try these queries to verify data collection:
     - `up` - Shows scrape status of all targets (1 = up, 0 = down)
     - `node_cpu_seconds_total` - CPU metrics from node exporters
     - `node_memory_MemFree_bytes` - Memory free on each host
     - `node_disk_io_time_seconds_total` - Disk I/O metrics
     - `rate(node_cpu_seconds_total[5m])` - CPU usage over 5 minutes

3. **Check current targets:**
   - Click "Status" → "Targets"
   - You should see node exporters from:
     - pi4-node1 (192.168.8.20:9100)
     - pi4-node2 (192.168.8.21:9100)
     - homelab-main (192.168.8.10:9100)
     - pi3-utils (192.168.8.22:9100)

### Configuration Issues to Fix

1. **Missing node exporter on pi3-utils:**

   - Check if `pi3-utils` is scraping at port 9100
   - If not, run: `ansible-playbook -i inventory.yml playbooks/setup_monitoring_stack.yml --tags node-exporter --limit pi3-utils -K`

2. **Router metrics (optional but useful):**

   - GL.iNet router (192.168.8.1) is not currently exposed to Prometheus
   - Would require installing node_exporter on OpenWrt or using collectd
   - Skip for now; add later if needed

3. **Update Prometheus config to add more scrape targets (optional):**
   - Jenkins metrics: Add `- targets: ['192.168.8.10:8080']` (if Jenkins exposes metrics)
   - Custom applications as they're added

### Retention and Performance

- Current: 7-day retention, 15-second scrape interval
- This is good for a homelab; adjust if storage becomes an issue
- To modify, edit the Prometheus ConfigMap in `setup_monitoring_stack.yml`

---

## 2. Grafana Setup

### Initial Login

- URL: http://192.168.8.20:30030
- Default credentials: `admin` / `admin`
- **⚠️ Change password on first login**

### Step 1: Verify Data Sources

1. **Log in and go to:**

   - Settings (⚙️) → Data Sources

2. **Verify existing sources:**

   - ✅ Prometheus (http://prometheus.monitoring.svc.cluster.local:9090)
   - ✅ Loki (http://loki.monitoring.svc.cluster.local:3100)

3. **Test connections:**
   - Click each data source
   - Click "Test" button - should show "Data source is working"

### Step 2: Create Essential Dashboards

#### Dashboard 1: System Overview (CPU, Memory, Disk)

1. **Create new dashboard:**

   - Click + → Dashboard

2. **Add panels with these queries:**

   **Panel 1: CPU Usage (all hosts)**

   ```
   Name: "CPU Usage %"
   Query (Prometheus):
   100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))
   Visualization: Graph
   ```

   **Panel 2: Memory Usage (all hosts)**

   ```
   Name: "Memory Used %"
   Query:
   100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
   Visualization: Gauge
   ```

   **Panel 3: Disk Usage (all hosts)**

   ```
   Name: "Disk Usage %"
   Query:
   100 * (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs|squashfs|vfat"} / node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs|squashfs|vfat"}))
   Visualization: Table (convert to table format)
   ```

   **Panel 4: System Uptime**

   ```
   Name: "Uptime (hours)"
   Query:
   node_boot_time_seconds
   Transformation: Convert field type (timestamp → time since now)
   ```

3. **Save dashboard:** Give it a name like "System Overview"

#### Dashboard 2: Per-Host Details

1. **Create new dashboard**
2. **Add variable for host selection:**

   - Settings → Variables → New Variable
   - Name: `host`
   - Query: `label_values(up, instance)`
   - Make it multi-select

3. **Add panels filtered by $host variable:**

   **CPU per core:**

   ```
   Query: rate(node_cpu_seconds_total{instance="$host",mode="system"}[5m])
   Legend: {{ cpu }}
   ```

   **Memory trend:**

   ```
   Query: node_memory_MemFree_bytes{instance="$host"}
   ```

   **Disk I/O:**

   ```
   Query: rate(node_disk_io_time_seconds_total{instance="$host"}[5m])
   ```

#### Dashboard 3: Network Monitoring (if metrics available)

```
Panel: "Network I/O"
Query: rate(node_network_receive_bytes_total{instance="$host",device!~"lo"}[5m])
```

### Step 3: Set Up Alerts (Optional for now)

1. **Go to:** Alerting → Alert rules
2. **Create a simple alert:**
   - CPU > 80% for 5 minutes
   - Memory > 85%
3. **Configure notification channel:** (Slack, email, etc.)

### Step 4: Configure Home Dashboard

1. **Settings** → **Home**
2. **Add starred dashboards** for quick access
3. **Set default home dashboard** to your "System Overview"

---

## 3. Pi-hole Configuration

### Initial Login

- URL: http://192.168.8.12:8080/admin
- Default password: `admin` (you set this in the playbook)
- **⚠️ Change password immediately**

### Step 1: Basic Configuration

#### Settings → General

- **DNS upstream servers:**

  - Primary: 1.1.1.1 (Cloudflare)
  - Secondary: 1.0.0.1 (Cloudflare)
  - ✅ Already set in playbook

- **DNS records for local hosts (optional):**
  - Go to Settings → Local DNS Records
  - Add entries for your homelab hosts:
    ```
    homelab-main.local    → 192.168.8.10
    pi4-node1.local       → 192.168.8.20
    pi4-node2.local       → 192.168.8.21
    homelab-mgmt.local    → 192.168.8.12
    grafana.local         → 192.168.8.20
    prometheus.local      → 192.168.8.20
    jenkins.local         → 192.168.8.10
    ```

### Step 2: Add Blocklists

1. **Go to:** Adlists (or Adlist)

2. **Add common blocklists:**

   | List Name             | URL                                                                         |
   | --------------------- | --------------------------------------------------------------------------- |
   | Steven Black          | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`          |
   | Firebog - Suspicious  | `https://raw.githubusercontent.com/firebogdan/hosts/master/suspicious.txt`  |
   | Firebog - Advertising | `https://raw.githubusercontent.com/firebogdan/hosts/master/advertising.txt` |
   | Firebog - Tracking    | `https://raw.githubusercontent.com/firebogdan/hosts/master/tracking.txt`    |
   | Firebog - Malware     | `https://raw.githubusercontent.com/firebogdan/hosts/master/malware.txt`     |
   | Pi-hole Regex         | `https://raw.githubusercontent.com/mmotti/pihole-regex/master/regex.list`   |

3. **Update gravity database:**
   - After adding lists, gravity updates automatically
   - Check Settings → Gravity to see blocklist stats

### Step 3: Configure DHCP

1. **Settings → DHCP Server**

2. **Enable DHCP server on Pi-hole:**

   - Toggle "DHCP server enabled"
   - This allows Pi-hole to serve DHCP to your network
   - Useful for ensuring all devices use Pi-hole as DNS

3. **Alternative: Configure router DHCP to use Pi-hole as DNS**
   - On GL.iNet router (192.168.8.1):
     - Network → LAN → DHCP Server
     - Set DNS 1: 192.168.8.12
     - Set DNS 2: 8.8.8.8 (backup)
   - This makes all devices use Pi-hole DNS automatically

### Step 4: Query Logging and Analytics

1. **Go to:** Settings → Query Logging

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
