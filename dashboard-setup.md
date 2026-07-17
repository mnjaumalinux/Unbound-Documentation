# Unbound Dashboard Setup

**Date:** 2026-07-17  
**Reference:** [ar51an/unbound-dashboard](https://github.com/ar51an/unbound-dashboard) v2.3  
**Server:** resolve01 (`102.219.210.213`)

---

## Stack Overview

| Component | Version | Role | Port |
|-----------|---------|------|------|
| Grafana OSS | 11.1.0 | Dashboard UI | 3000 |
| Prometheus | 2.45+ (apt) | Metrics time-series DB | 9090 |
| unbound-exporter | ar51an/unbound-exporter (compiled amd64) | Scrapes Unbound stats via Unix socket | 9167 |
| Loki | 3.1.0 | Log aggregation | 3100 |
| Promtail | 3.1.0 | Log shipper | 9080 |

**Dashboard URL:** `http://102.219.210.213:3000/d/ee245028-3ff4-444f-a43f-b2c021346cd9/unbound`  
**Login:** `admin` / *(set at install)*

---

## Architecture

```
Unbound DNS
  │
  ├─── Unix socket (/run/unbound.ctl) ───► unbound-exporter :9167 ───► Prometheus :9090
  │                                                                            │
  └─── Log file (/var/log/unbound/unbound.log) ───► Promtail :9080 ──► Loki :3100
                                                                               │
                                                                         Grafana :3000
                                                                        (25-panel dashboard)
```

---

## Installation Summary

### 1. Unbound configuration additions

Two drop-in config files were added:

**`/etc/unbound/unbound.conf.d/dashboard.conf`**
```yaml
server:
    extended-statistics: yes
    verbosity: 0
    log-queries: yes
    log-replies: yes
    log-tag-queryreply: yes
    log-local-actions: yes
    logfile: "/var/log/unbound/unbound.log"

remote-control:
    control-enable: yes
    control-interface: "/run/unbound.ctl"
    control-use-cert: no
```

> **Note:** AppArmor only permits `/run/unbound.ctl` as the Unix socket path and `/var/log/unbound/` for log writes. A local AppArmor override was added at `/etc/apparmor.d/local/usr.sbin.unbound`.

**`/etc/tmpfiles.d/unbound.conf`** — ensures `/run/unbound/` directory persists across reboots:
```
d /run/unbound 0755 unbound unbound -
```

### 2. Log directory and rotation

```bash
sudo mkdir -p /var/log/unbound
sudo chown unbound:unbound /var/log/unbound
```

Logrotate config: `/etc/logrotate.d/unbound` (daily, 7 days, compress)

### 3. unbound-exporter (compiled from source for amd64)

The release binary is arm64-only. Compiled from source:

```bash
sudo apt install golang-go git
cd /tmp && git clone https://github.com/ar51an/unbound-exporter.git
cd unbound-exporter && go mod tidy && go build && strip unbound-exporter
sudo cp unbound-exporter /usr/local/bin/
```

Service: `/etc/systemd/system/prometheus-unbound-exporter.service`
```ini
[Unit]
Description=Prometheus Unbound Exporter
After=network.target unbound.service

[Service]
User=root
ExecStart=/usr/local/bin/unbound-exporter --unbound.uri unix:///run/unbound.ctl
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 4. Grafana OSS (amd64)

```bash
sudo apt install musl
wget https://dl.grafana.com/oss/release/grafana_11.1.0_amd64.deb
sudo dpkg -i grafana_11.1.0_amd64.deb
sudo systemctl enable --now grafana-server
```

### 5. Prometheus

```bash
sudo apt install prometheus
sudo apt --purge autoremove prometheus-node-exporter
```

Config: `/etc/prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 5m
  evaluation_interval: 5m

scrape_configs:
  - job_name: 'unbound'
    static_configs:
      - targets: ['localhost:9167']
```

### 6. Loki + Promtail (amd64)

```bash
curl -LO https://github.com/grafana/loki/releases/download/v3.1.0/loki_3.1.0_amd64.deb
curl -LO https://github.com/grafana/loki/releases/download/v3.1.0/promtail_3.1.0_amd64.deb
sudo dpkg -i loki_3.1.0_amd64.deb promtail_3.1.0_amd64.deb
```

Configs used from official release v2.3 tarball (adapted for amd64 paths):
- `/etc/loki/config.yml` — optimised for large Unbound log datasets
- `/etc/promtail/config.yml` — scrapes `/var/log/unbound/unbound.log`, labels `dns: reply` and `dns: block`

---

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/unbound/unbound.conf.d/dashboard.conf` | Extended stats, Unix socket, file logging |
| `/etc/apparmor.d/local/usr.sbin.unbound` | AppArmor allow for log dir and socket |
| `/etc/tmpfiles.d/unbound.conf` | Persist `/run/unbound/` dir across reboots |
| `/var/log/unbound/unbound.log` | Unbound query log (Promtail source) |
| `/etc/logrotate.d/unbound` | Daily rotation, 7-day retention |
| `/usr/local/bin/unbound-exporter` | Prometheus exporter binary (amd64) |
| `/etc/systemd/system/prometheus-unbound-exporter.service` | Exporter service unit |
| `/etc/prometheus/prometheus.yml` | Prometheus scrape config |
| `/etc/loki/config.yml` | Loki storage and query config |
| `/etc/promtail/config.yml` | Promtail log scrape and labelling |
| `/etc/grafana/grafana.ini` | Grafana server config |

---

## Service Management

```bash
# Status of all services
for svc in grafana-server prometheus prometheus-unbound-exporter loki promtail unbound; do
    printf "%-35s %s\n" "$svc" "$(systemctl is-active $svc)"
done

# Restart individual services
sudo systemctl restart grafana-server
sudo systemctl restart prometheus
sudo systemctl restart prometheus-unbound-exporter
sudo systemctl restart loki
sudo systemctl restart promtail
sudo systemctl restart unbound

# View logs
sudo journalctl -u prometheus-unbound-exporter -f
sudo journalctl -u loki -f
sudo journalctl -u promtail -f
sudo tail -f /var/log/unbound/unbound.log
```

---

## Verification

```bash
# Metrics endpoint
curl localhost:9167/metrics | grep unbound_answer_rcodes

# Loki log ingestion
curl -sG "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="unbound"}' \
  --data-urlencode "limit=5"

# Prometheus target health
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health

# Grafana API check
curl -s -u admin:<password> http://localhost:3000/api/datasources | python3 -m json.tool | grep name
```

---

## Known Issues & Fixes Applied

| Issue | Fix |
|-------|-----|
| AppArmor blocked `/var/run/unbound.sock` | Used AppArmor-permitted path `/run/unbound.ctl` |
| AppArmor blocked `/var/log/unbound/` writes | Added `/etc/apparmor.d/local/usr.sbin.unbound` override |
| Release binary is arm64-only | Compiled `unbound-exporter` from Go source for amd64 |
| apt mirror `ke.archive.ubuntu.com` broken | Switched to `archive.ubuntu.com` in `/etc/apt/sources.list.d/ubuntu.sources` |
| Grafana default `admin/admin` rejected | Reset via `sudo grafana-cli admin reset-admin-password` |

---

## Dashboard Panels (25 total)

The dashboard provides metrics and log views including:
- Query rate (total, cache hits, cache misses)
- Cache hit rate over time
- Response codes (NOERROR, SERVFAIL, NXDOMAIN, REFUSED)
- DNSSEC validation status (secure, bogus)
- Blocklist hits
- Top queried domains (from Loki)
- DNS query log stream (live, from Loki)
- unbound-exporter status panel
