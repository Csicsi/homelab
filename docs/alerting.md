# Alerting and Notifications

## Goal

Get notified when something breaks without needing a full multi-node monitoring stack.

---

## Recommended Baseline

Run these on the homelab laptop first:

- **Uptime Kuma** for HTTP, TCP, ping, and keyword checks
- **Prometheus + Grafana** for host and container metrics
- **Node Exporter** on the laptop and optional monitoring Pi

If you keep the Pi online, run at least one independent checker there so it can still alert when the laptop is down.

---

## Simple Notification Path

### Option 1: Uptime Kuma + ntfy

- Easy to self-host
- Push notifications to phone and desktop
- Minimal setup overhead

Suggested use:

- Uptime Kuma on the laptop for most service checks
- Optional second Uptime Kuma instance on the Pi checking the laptop
- Send alerts to a private ntfy topic

### Option 2: Grafana Alerting

Use Grafana alerts for:

- High CPU or memory usage
- Low disk space
- Node Exporter missing from scrape targets
- Prometheus target down

Send notifications to:

- Email
- Discord
- Telegram
- Webhook endpoints

### Option 3: Mixed Setup

Best overall balance:

- **Uptime Kuma** for endpoint and availability alerts
- **Grafana Alerting** for metrics-based alerts

---

## What to Monitor First

Start with a very small set of high-value checks:

1. Laptop reachable by ping
2. Router reachable by ping
3. Internet reachability from the Pi or router-side checks
4. Docker daemon running on the laptop
5. Key services responding on their HTTP ports
6. Disk usage on the laptop above a threshold

---

## Redundancy Pattern

If the Pi stays in the setup, use it as the independent watcher:

- Laptop runs the main dashboards and local monitoring stack
- Pi runs a lightweight alert path such as Uptime Kuma or ntfy relay
- Pi checks the laptop, router, and internet connectivity

This is usually enough redundancy for a small homelab.

---

## Recommendation

For this repo, the simplest target state is:

- Laptop: Docker, Prometheus, Grafana, Uptime Kuma, Homer/Portainer
- Pi: optional Uptime Kuma or secondary Prometheus/exporter path
- Router: WireGuard and DHCP only

That gives visibility, notifications, and a small operational footprint.
