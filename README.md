# Multi-VPS Infrastructure Monitoring with Grafana Cloud & Alloy

This repository documents the setup and configuration for aggregating system metrics from multiple client VPS instances into a single, centralized **Grafana Cloud (Free Tier)** dashboard using **Grafana Alloy** and **Prometheus**.

---

## 🏗️ Architecture Overview

Instead of running a heavy, resource-consuming Prometheus server on each client VPS, we utilize **Grafana Alloy** as a lightweight telemetry collector.

### Data Flow

1. **Alloy's Embedded Unix Exporter** scrapes local host metrics:
   - CPU
   - Memory
   - Disk
   - Network

2. A **Relabeling Pipeline** injects a custom, human-readable server name onto the collected metrics.

3. Alloy securely streams metrics via **Prometheus Remote Write** to the hosted Grafana Cloud instance.

```text
┌─────────────────────────┐
│      Client VPS         │
│                         │
│  Unix Exporter          │
│        ↓                │
│  Prometheus Scrape      │
│        ↓                │
│  Relabel Pipeline       │
│        ↓                │
│  Remote Write           │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│     Grafana Cloud       │
│                         │
│  Prometheus Storage     │
│        ↓                │
│    Dashboards           │
└─────────────────────────┘
```

---

# 🚀 Step-by-Step Installation & Setup

Follow these steps for every new VPS instance you want to bring under monitoring.

---

## 1. Install Grafana Alloy (Ubuntu/Debian)

Log into your client VPS via SSH and run the following commands to add the Grafana repository and install the Alloy agent.

```bash
# Add the Grafana repository GPG key
sudo mkdir -p /etc/apt/keyrings

wget -q -O - https://apt.grafana.com/gpg.key \
| sudo gpg --dearmor \
| sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add the Alloy repository to apt sources
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" \
| sudo tee /etc/apt/sources.list.d/grafana.list

# Update package lists and install Alloy
sudo apt-get update
sudo apt-get install alloy -y
```

---

## 2. Configure the Monitoring Agent

Open the configuration file using a text editor:

```bash
sudo nano /etc/alloy/config.alloy
```

Wipe out any boilerplate code and replace it with the configuration below.

```hcl
// 1. Enable the local Linux system metrics collector
prometheus.exporter.unix "local_system" {
}

// 2. Scrape system metrics every 30 seconds
prometheus.scrape "scrape_system_metrics" {
  targets = prometheus.exporter.unix.local_system.targets

  // CRITICAL:
  // Forward to the relabel block FIRST,
  // not directly to the cloud.
  forward_to = [prometheus.relabel.add_vps_label.receiver]

  scrape_interval = "30s"
}

// 3. Inject unique identification labels per VPS
prometheus.relabel "add_vps_label" {

  // Forward the newly labeled data
  // to the remote write client.
  forward_to = [prometheus.remote_write.grafana_cloud.receiver]

  rule {
    target_label = "instance"

    replacement = "client-vps-1"
    // MUST CHANGE FOR EACH VPS
    // e.g. client-vps-2, client-vps-3

    action = "replace"
  }
}

// 4. Authenticate and ship metrics to Grafana Cloud
prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = "YOUR_GRAFANA_CLOUD_REMOTE_WRITE_URL/api/prom/push"

    basic_auth {
      username = "YOUR_GRAFANA_CLOUD_USERNAME"
      password = "YOUR_GRAFANA_CLOUD_API_TOKEN"
    }
  }
}
```

---

## 3. Restart and Enable the Service

Whenever you edit the configuration file, restart the Alloy daemon so it can reload the pipeline.

```bash
sudo systemctl restart alloy
sudo systemctl enable alloy
```

---

# ⚠️ Crucial Things to Look Out For (Gotchas)

During the initial deployment phase, pay close attention to these common points of failure.

---

## 1. The Unique `instance` Label (Multi-Server Dropdown)

Inside the `prometheus.relabel "add_vps_label"` block, you **must** change the `replacement` value for every VPS you provision.

### Example

```hcl
# VPS 1
replacement = "client-vps-1"

# VPS 2
replacement = "client-vps-2"

# VPS 3
replacement = "client-vps-3"
```

### Why This Matters

If multiple servers stream metrics using the same `instance` label:

- Grafana Cloud sees them as the same host.
- Metrics arrive with conflicting timestamps.
- Samples become out-of-order.
- Data gets discarded.
- Monitoring becomes unreliable.

---

## 2. The Remote Write URL Path (`/api/prom/push`)

When copying your Prometheus endpoint from Grafana Cloud Instance Details, ensure the URL ends with:

```text
/api/prom/push
```

### Incorrect

```text
https://prometheus-prod-instance.grafana.net/api/prom
```

### Correct

```text
https://prometheus-prod-instance.grafana.net/api/prom/push
```

### Why This Matters

Omitting `/push` causes Alloy to target the wrong endpoint and results in:

- HTTP 404 errors
- Silent telemetry failures
- Metrics never reaching Grafana Cloud

---

# 🔍 Troubleshooting & Verification

If metrics are not appearing in Grafana dashboards, use the following methods to verify that Alloy is operating correctly.

---

## Inspecting Live Engine Logs

To inspect Alloy logs without entering an interactive pager:

```bash
sudo journalctl -u alloy --no-pager -n 20
```

### Flag Explanation

| Flag | Purpose |
|--------|----------|
| `--no-pager` | Prints logs directly to the terminal instead of opening a pager |
| `-n 20` | Shows only the most recent 20 log entries |

---

## Healthy Log Examples

```text
level=info msg="starting cluster node" service=cluster
level=info msg="now listening for http traffic" service=http addr=127.0.0.1:12345
level=info msg="Done replaying WAL" component_id=prometheus.remote_write.grafana_cloud
```

If you see:

```text
Done replaying WAL
```

without any subsequent errors, Alloy is successfully forwarding metrics to Grafana Cloud.

---

## Common Error Patterns

### HTTP 404 Not Found

```text
HTTP status 404 Not Found
```

**Cause:**

The remote write URL is incorrect.

**Fix:**

Verify that the URL ends with:

```text
/ api/prom/push
```

(remove the space when using it)

---

### HTTP 401 Unauthorized

```text
HTTP status 401 Unauthorized
```

**Cause:**

Authentication credentials are incorrect.

**Fix:**

Verify:

- Grafana Cloud Username / Instance ID
- Grafana Cloud API Token

---

### Out-of-Order Samples

```text
non-recoverable error ... out of order sample
```

**Cause:**

Metrics are being forwarded directly from the scrape stage to remote write, bypassing the relabel stage.

**Fix:**

Ensure your scrape block forwards to:

```hcl
forward_to = [prometheus.relabel.add_vps_label.receiver]
```

and **not** directly to:

```hcl
prometheus.remote_write.grafana_cloud.receiver
```

---

# 📊 Visualizing Inside Grafana Cloud

Once metrics are flowing successfully:

1. Log into your Grafana Cloud workspace.
2. Navigate to:

   ```text
   Dashboards → New → Import
   ```

3. Import the community dashboard:

   ```text
   Dashboard ID: 1860
   ```

   **Node Exporter Full**

4. Select your Grafana Cloud Prometheus data source.

5. Apply the dashboard.

---

## Expected Result

You should now be able to:

- Monitor multiple VPS instances from a single dashboard.
- Filter servers using the `instance` dropdown.
- View:
  - CPU usage
  - Memory consumption
  - Disk utilization
  - Network throughput
  - Load averages
  - Filesystem statistics

All from a single Grafana Cloud Free Tier deployment.