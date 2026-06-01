# Grafana Alloy Setup Guide (Caddy & CloudPanel)

This guide outlines the configuration steps required to deploy Grafana Alloy on servers using a **Caddy-first reverse proxy** running upstream of **CloudPanel (Nginx)**, as well as an alternative standalone **Caddy-only** architecture.

---

# 🏗️ Architecture Overview

## Option A: Dual Proxy (Current Setup)

### Components

- **Caddy (Ports 80/443)**
  - Handles public traffic
  - Manages global On-Demand TLS
  - Reverse proxies requests to CloudPanel

- **CloudPanel Nginx (Port 8081)**
  - Hosts websites and applications
  - Generates application access logs

- **Grafana Alloy**
  - Collects hardware metrics
  - Scrapes Caddy operational metrics
  - Ships Nginx access logs to Grafana Loki

### Data Flow

```text
Internet
    │
    ▼
┌─────────────┐
│    Caddy    │
│ 80 / 443    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ CloudPanel  │
│ Nginx 8081  │
└──────┬──────┘
       │
       ▼
 Applications

        ▲
        │ Logs
        │
┌───────┴────────┐
│ Grafana Alloy  │
└───────┬────────┘
        │
        ▼
 Grafana Cloud
 (Prometheus + Loki)
```

---

## Option B: Standalone Proxy (Alternative Setup)

### Components

- **Caddy (Ports 80/443)**
  - Handles traffic
  - Manages TLS certificates
  - Serves applications directly

- **Grafana Alloy**
  - Collects hardware metrics
  - Scrapes Caddy metrics
  - Ships Caddy access logs directly to Loki

### Data Flow

```text
Internet
    │
    ▼
┌─────────────┐
│    Caddy    │
│ 80 / 443    │
└──────┬──────┘
       │
       ▼
 Applications

       ▲
       │ Logs
       │
┌──────┴──────┐
│    Alloy    │
└──────┬──────┘
       │
       ▼
 Grafana Cloud
```

---

# 🚀 Deployment Steps (Option A: Dual Proxy Setup)

---

## 1. Update the Caddy Configuration

Ensure Caddy's internal Admin API is reachable locally.

Edit:

```text
/etc/caddy/Caddyfile
```

Global configuration:

```caddy
{
    on_demand_tls {
        ask http://127.0.0.1:8081/api/is-domain-valid
        interval 2m
        burst 5
    }
}

:443, :80 {
    tls {
        on_demand
    }

    reverse_proxy 127.0.0.1:8081 {
        header_up Host {http.request.host}
        header_up X-Real-IP {http.request.remote.host}
    }
}
```

> Do not add any standalone metrics directives. The Admin API already exposes metrics on port `2019`.

### Reload and Verify

```bash
sudo systemctl restart caddy
```

Verify metrics are exposed:

```bash
curl http://127.0.0.1:2019/metrics
```

You should see Prometheus-formatted output.

---

## 2. Set Up Log Access Permissions

Grafana Alloy runs as a restricted system user and requires read access to CloudPanel log files.

Grant permissions using ACLs:

```bash
sudo setfacl -R -m u:alloy:rx /home/clp-user/logs
sudo setfacl -dR -m u:alloy:rx /home/clp-user/logs
```

### Verify

```bash
getfacl /home/clp-user/logs
```

You should see an entry similar to:

```text
user:alloy:r-x
```

---

## 3. Apply the Alloy Configuration

Replace the contents of:

```text
/etc/alloy/config.river
```

with the following configuration.

---

### Metrics Collection

```river
// ==========================================
// 1. METRICS COLLECTION
// ==========================================

// Hardware Stats (CPU, RAM, Disk)
prometheus.exporter.unix "local_system" {}

prometheus.scrape "scrape_system_metrics" {
  targets         = prometheus.exporter.unix.local_system.targets
  forward_to      = [prometheus.relabel.add_vps_label.receiver]
  scrape_interval = "30s"
}

// Caddy Proxy Metrics
prometheus.scrape "caddy" {
  targets = [
    {
      "__address__" = "127.0.0.1:2019"
    }
  ]

  forward_to = [prometheus.relabel.add_vps_label.receiver]
}
```

---

### Log Collection (Loki)

```river
// ==========================================
// 2. LOG COLLECTION (LOKI)
// ==========================================

// CloudPanel / Nginx Logs
local.file_match "cloudpanel_logs" {
  path_targets = [
    { "__path__" = "/home/clp-user/logs/*.log" },
    { "__path__" = "/var/log/nginx/*.log" }
  ]
}

loki.source.file "log_scrape" {
  targets    = local.file_match.cloudpanel_logs.targets
  forward_to = [loki.write.grafana_loki.receiver]
}
```

---

### Relabeling Pipeline

```river
// ==========================================
// 3. PIPELINE RELABELING
// ==========================================

prometheus.relabel "add_vps_label" {
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]

  rule {
    target_label = "instance"

    replacement = "your-caddy-cloudpanel-vps-name"

    action = "replace"
  }
}
```

> Change the `replacement` value to a unique server name for every VPS.

---

### Grafana Cloud Export

```river
// ==========================================
// 4. DATA EXPORT
// ==========================================

prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = "https://<your-prometheus-url>/api/prom/push"

    basic_auth {
      username = "YOUR_PROMETHEUS_USER_ID"
      password = "YOUR_GRAFANA_CLOUD_API_TOKEN"
    }
  }
}

loki.write "grafana_loki" {
  endpoint {
    url = "https://<your-loki-url>/loki/api/v1/push"

    basic_auth {
      username = "YOUR_LOKI_USER_ID"
      password = "YOUR_GRAFANA_CLOUD_API_TOKEN"
    }
  }
}
```

---

## 4. Restart Alloy

```bash
sudo systemctl restart alloy
```

Watch the logs live:

```bash
sudo journalctl -u alloy -f -n 20
```

---

# 🔍 Verification

### Verify Caddy Metrics

```bash
curl http://127.0.0.1:2019/metrics
```

Expected:

```text
caddy_admin_http_requests_total
caddy_http_requests_total
...
```

---

### Verify Alloy Status

```bash
sudo systemctl status alloy
```

Expected:

```text
active (running)
```

---

### Verify Recent Alloy Logs

```bash
sudo journalctl -u alloy --no-pager -n 50
```

Healthy example:

```text
level=info msg="Done replaying WAL"
level=info msg="Successfully sent batch"
```

---

# ⚠️ Common Issues

## Caddy Metrics Not Available

### Error

```text
connection refused 127.0.0.1:2019
```

### Check

```bash
sudo systemctl status caddy
```

### Verify

```bash
curl http://127.0.0.1:2019/config/
```

---

## Alloy Cannot Read Log Files

### Error

```text
permission denied
```

### Fix

```bash
sudo setfacl -R -m u:alloy:rx /home/clp-user/logs
sudo setfacl -dR -m u:alloy:rx /home/clp-user/logs
```

---

## Prometheus Authentication Failure

### Error

```text
401 Unauthorized
```

### Verify

- Prometheus Username
- Loki Username
- Grafana Cloud API Token

---

## Metrics Missing from Dashboard

### Common Causes

- Duplicate `instance` labels
- Incorrect `/api/prom/push` endpoint
- Failed relabel stage
- Firewall restrictions
- Invalid Grafana Cloud credentials

---

# 🔄 Alternative Setup (Option B: Standalone Caddy)

If you remove CloudPanel and run Caddy as the sole web server, use the following modifications.

---

## 1. Enable Structured Logging in Caddy

Edit:

```text
/etc/caddy/Caddyfile
```

Example:

```caddy
{
    on_demand_tls {
        ask http://127.0.0.1:8000/api/is-domain-valid
    }
}

example.com {

    log {
        output file /var/log/caddy/access.log {
            roll_size 10mb
            roll_keep 5
        }

        format json
    }

    reverse_proxy 127.0.0.1:8000
}
```

---

## 2. Create Log Directory

```bash
sudo mkdir -p /var/log/caddy
sudo chown -R caddy:caddy /var/log/caddy
```

---

## 3. Grant Alloy Access

```bash
sudo setfacl -R -m u:alloy:rx /var/log/caddy
sudo setfacl -dR -m u:alloy:rx /var/log/caddy
```

---

## 4. Modify Alloy Log Collection

Replace the previous CloudPanel log collector with:

```river
// ==========================================
// 2. STANDALONE LOG COLLECTION (CADDY DIRECT)
// ==========================================

local.file_match "caddy_logs" {
  path_targets = [
    { "__path__" = "/var/log/caddy/*.log" }
  ]
}

loki.source.file "log_scrape" {
  targets    = local.file_match.caddy_logs.targets
  forward_to = [loki.write.grafana_loki.receiver]
}
```

---

# 📊 Grafana Cloud Outcome

Once configured successfully, Grafana Cloud will provide:

### Metrics

- CPU Usage
- Memory Usage
- Disk Utilization
- Network Throughput
- Filesystem Health
- Caddy Operational Metrics

### Logs

- CloudPanel Nginx Access Logs
- CloudPanel Error Logs
- Standalone Caddy Access Logs
- Structured JSON Request Data

### Benefits

- Single-pane observability
- Centralized multi-server monitoring
- Log search and retention
- Infrastructure alerting
- Lightweight VPS footprint
- No self-hosted Prometheus or Loki maintenance