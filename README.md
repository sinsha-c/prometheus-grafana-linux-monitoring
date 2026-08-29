# Linux Server Monitoring with Prometheus & Grafana

A hands-on DevOps lab that implements a complete monitoring and alerting solution for a Linux server — from metric collection with **Prometheus** and **Node Exporter**, to visualization and alerting with **Grafana**.

Part of my `#AWSDevOpsRestartJourney` — rebuilding and documenting real-world DevOps skills, one project at a time.

---

## Overview

This project simulates a common real-world DevOps responsibility: keeping visibility into server health. It covers the full monitoring lifecycle — collecting system metrics, visualizing them on dashboards, and triggering alerts when thresholds are breached.

**What this lab demonstrates:**
- Installing and configuring Prometheus as a metrics collection and storage system
- Exposing Linux system metrics using Node Exporter
- Building Grafana dashboards for CPU, memory, disk, and network monitoring
- Writing PromQL queries to extract meaningful metrics
- Configuring threshold-based alert rules and notifications
- End-to-end testing of the alerting pipeline

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **Linux (Ubuntu)** | Host server being monitored |
| **Prometheus** | Metrics collection & time-series storage |
| **Node Exporter** | Exposes host-level metrics (CPU, memory, disk, network) |
| **Grafana** | Dashboards & alert notifications |
| **PromQL** | Query language for Prometheus metrics |

---

## Architecture

<img src="screenshots/architecture-diagram.png" alt="Architecture diagram: Node Exporter to Prometheus to Grafana to Alert Channel" width="750">

**Flow:** Node Exporter collects host metrics → Prometheus scrapes and stores them → Grafana queries Prometheus (via PromQL) to render dashboards and evaluate alert rules → notifications fire when thresholds are crossed.

---

## Prerequisites

Before starting, make sure you have:

- A Linux server (Ubuntu 20.04/22.04 recommended) with `sudo` access — a local VM, EC2 instance, or any cloud VM works
- At least **1 vCPU / 1 GB RAM** free (2 GB+ recommended so Prometheus, Node Exporter, and Grafana can run comfortably together)
- **Internet access** on the server to download the Prometheus, Node Exporter, and Grafana packages
- Basic familiarity with the Linux command line (`sudo`, `systemctl`, editing files with `nano`/`vim`)
- The following ports open in your firewall / security group:

  | Port | Used by |
  |---|---|
  | `9090` | Prometheus UI |
  | `9100` | Node Exporter metrics endpoint |
  | `3000` | Grafana UI |

- `wget`, `tar`, and `curl` installed (usually available by default; install with `sudo apt-get install -y wget tar curl` if missing)
- An email address or Slack webhook to use as the alert notification destination (needed for Step 9)

---

## Steps Performed

### 1. Install & Configure Prometheus

- Downloaded and installed Prometheus on the Linux server:

  ```bash
  # Create a dedicated user for Prometheus
  sudo useradd --no-create-home --shell /bin/false prometheus

  # Download and extract the latest Prometheus release
  wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
  tar -xvf prometheus-2.53.0.linux-amd64.tar.gz
  cd prometheus-2.53.0.linux-amd64

  # Move binaries and config to standard locations
  sudo mv prometheus promtool /usr/local/bin/
  sudo mkdir -p /etc/prometheus /var/lib/prometheus
  sudo mv prometheus.yml /etc/prometheus/
  sudo mv consoles/ console_libraries/ /etc/prometheus/

  # Set ownership
  sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
  ```

- The default `prometheus.yml` (moved into place above) already includes a scrape job for Prometheus itself, which is enough to start the service. The **Node Exporter scrape job** is added later, in **Step 3**, once Node Exporter is installed.

- Started Prometheus as a systemd service for reliability. Create the service file below now — this is separate from `prometheus.yml`: the service file tells systemd *how to run* Prometheus, while `prometheus.yml` tells Prometheus *what to scrape*:

  ```ini
  # /etc/systemd/system/prometheus.service
  [Unit]
  Description=Prometheus
  Wants=network-online.target
  After=network-online.target

  [Service]
  User=prometheus
  ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/

  [Install]
  WantedBy=multi-user.target
  ```

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable prometheus
  sudo systemctl start prometheus
  sudo systemctl status prometheus
  ```

- Verified the Prometheus UI on port `9090` by opening `http://<server-ip>:9090` in a browser.

<img src="screenshots/01-prometheus-service-running.png" alt="Prometheus systemd service running and UI loaded on port 9090" width="700">

### 2. Install & Configure Node Exporter

- Installed Node Exporter to expose OS-level metrics (CPU, memory, disk, network):

  ```bash
  # Create a dedicated user for Node Exporter
  sudo useradd --no-create-home --shell /bin/false node_exporter

  # Download and extract the latest release
  wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
  tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz

  # Move the binary to a standard location
  sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
  sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
  ```

- Ran Node Exporter as a systemd service on port `9100`:

  ```ini
  # /etc/systemd/system/node_exporter.service
  [Unit]
  Description=Node Exporter
  Wants=network-online.target
  After=network-online.target

  [Service]
  User=node_exporter
  ExecStart=/usr/local/bin/node_exporter

  [Install]
  WantedBy=multi-user.target
  ```

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable node_exporter
  sudo systemctl start node_exporter
  sudo systemctl status node_exporter
  ```

- Verified that metrics were being exposed:

  ```bash
  curl http://public-ip:9100/metrics | head -20
  ```

<img src="screenshots/02-node-exporter-metrics.png" alt="Node Exporter metrics endpoint output on port 9100" width="700">

### 3. Configure Prometheus to Scrape Node Exporter

- Added a scrape job for `node_exporter` in `prometheus.yml`:

  ```yaml
  # /etc/prometheus/prometheus.yml
  global:
    scrape_interval: 15s
    evaluation_interval: 15s

  scrape_configs:
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']

    - job_name: 'node_exporter'
      static_configs:
        - targets: ['localhost:9100']
  ```

- Validated the config file and reloaded Prometheus:

  ```bash
  promtool check config /etc/prometheus/prometheus.yml
  sudo systemctl restart prometheus
  ```

- Confirmed the target shows as **UP** under `Status → Targets` in the Prometheus UI (`http://<server-ip>:9090/targets`).

<img src="screenshots/03-prometheus-targets-up.png" alt="Prometheus targets page showing Node Exporter as UP" width="700">

### 4. Install & Configure Grafana

- Installed Grafana on the same server using the official Ubuntu/Debian APT repository:

  ```bash
  sudo apt-get install -y apt-transport-https software-properties-common wget

  # Add Grafana's GPG key and APT repository
  sudo mkdir -p /etc/apt/keyrings/
  wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
  echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

  sudo apt-get update
  sudo apt-get install -y grafana
  ```

- Started and enabled the Grafana service:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable grafana-server
  sudo systemctl start grafana-server
  sudo systemctl status grafana-server
  ```

- Logged into the Grafana UI on port `3000`:
  - Open `http://<server-ip>:3000`
  - Default credentials: `admin` / `admin` (you'll be prompted to change the password on first login)

<img src="screenshots/04-grafana-login.png" alt="Grafana login page and home dashboard on port 3000" width="700">

- After loggedin

<img src="screenshots/04-grafana-home.png" alt="Grafana login page and home dashboard on port 3000" width="700">

### 5. Add Prometheus as a Data Source

- Connected Grafana to Prometheus:
  1. In Grafana, go to **Connections → Data sources → Add data source**.
  2. Select **Prometheus**.
  3. Set the URL to `http://localhost:9090`.
  4. Leave the authentication settings as default (no auth is needed for a local setup).
  5. Click **Save & Test**.

- Verified that the connection was successful — Grafana shows a green **"Successfully queried the Prometheus API"** message.

<img src="screenshots/05-grafana-datasource-connected.png" alt="Grafana Prometheus data source connection successful" width="700">

### 6. Build the Monitoring Dashboard

Created a Grafana dashboard with panels for CPU, memory, disk, and network:

1. Go to **Dashboards → New → New Dashboard **.
2. Click **Add Panel and Add visualization** and select the **Prometheus** data source.
3. For each panel:
   - Paste the relevant PromQL query (see Step 7).
   - Set the panel title (e.g. "CPU Utilization").
   - Choose a visualization type — **Time series** works well for CPU/Memory/Network, and **Gauge** works well for Disk.
   - Set the unit to **Percent (0–100)** for the CPU/Memory/Disk panels.
   > <img src="screenshots/06-grafana-dashboard-set-panel.png" width="700">

4. Repeat for all four panels:
   - **CPU Utilization**
   - **Memory Utilization**
   - **Disk Utilization**
   - **Network Traffic**
5. Click **Save dashboard**, give it a name (e.g. "Linux Server Monitoring"), and save.
  > <img src="screenshots/06-grafana-dashboard-save.png" width="700">

Grafana dashboard with CPU, Memory, Disk and Network panels

<img src="screenshots/06-grafana-dashboard-overview.png" alt="Grafana dashboard with CPU, Memory, Disk and Network panels" width="700">

### 7. PromQL Queries Used

Each query below was entered directly into the panel's query editor (Step 6 → panel → Query tab):

| Metric | PromQL Query |
|---|---|
| CPU Usage | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| Memory Usage | `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100` |
| Disk Usage | `100 - ((node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100)` |
| Network Traffic (Receive) | `rate(node_network_receive_bytes_total{device!="lo"}[5m])` |
| Network Traffic (Transmit) | `rate(node_network_transmit_bytes_total{device!="lo"}[5m])` |

<img src="screenshots/07-promql-query-editor.png" alt="PromQL query editor in Grafana showing a metric query" width="700">

### 8. Configure Alert Rules

Set up alert rules in Grafana for critical thresholds:

1. Go to **Alerting → Alert rules → New alert rule**.
2. **CPU alert:**
   - Query: `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
   - Condition: `IS ABOVE 80`
   - Evaluation: evaluate every `1m`, for `5m` (fires only after the condition holds for 5 minutes)
   - Name: `High CPU Usage`
3. **Disk alert:**
   - Query: `100 - ((node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100)`
   - Condition: `IS ABOVE 80`
   - Evaluation: evaluate every `1m`, for `5m`
   - Name: `High Disk Usage`
4. Add a summary/description annotation to each rule (e.g. `CPU usage is above 80% for more than 5 minutes`).
5. Assign both rules to a folder (e.g. `Server Monitoring`) and save.

**Alert rules configured:**
- CPU usage above 80% for 5 minutes
- Disk usage above 80% for 5 minutes

<img src="screenshots/08-alert-rule-configuration.png" alt="Grafana alert rule configuration for CPU and disk thresholds" width="700">

### 9. Configure Alert Notifications

- Created a Grafana contact point (notification channel):
  1. Go to **Alerting → Contact points → Add contact point**.
  2. Name it (e.g. `email-alerts`) and choose a channel type — **Email** (or Slack/webhook).
  3. For email, enter the recipient address (SMTP must be configured in `/etc/grafana/grafana.ini` under `[smtp]`, or use a service such as Gmail SMTP or Mailgun).
  4. Click **Test** to send a sample notification, then **Save**.

- Linked the contact point to the alert rules:
  1. Go to **Alerting → Notification policies**.
  2. Edit the default policy (or add a nested policy) and set **Contact point** to `email-alerts`.
  3. Optionally match by label (e.g. `severity=critical`) to route only specific alerts.

- Verified that the notification policy routes alerts correctly using the **Test** button on the contact point.

<img src="screenshots/09-contact-point-notification-policy.png" alt="Grafana contact point and notification policy configuration" width="700">

### 10. Test the Alerting Pipeline

- Generated artificial CPU load to cross the CPU threshold:

  ```bash
  sudo apt-get install -y stress
  stress --cpu 4 --timeout 360s   # keeps 4 CPU cores busy for 6 minutes
  ```

- Filled disk space temporarily to cross the disk threshold:

  ```bash
  fallocate -l 5G /tmp/fill_disk.img   # adjust the size based on available disk space

  # remove afterwards to free space:
  rm /tmp/fill_disk.img
  ```

- Watched the alert state transition in **Alerting → Alert rules**:
  - **Normal** → **Pending** (condition breached, waiting out the `for: 5m` duration) → **Firing** (notification sent)

- Confirmed that the notification was received successfully in the configured email inbox or Slack channel.

<img src="screenshots/10-alert-firing-state.png" alt="Grafana alert showing Firing state after threshold breach" width="700">
<img src="screenshots/11-notification-received.png" alt="Alert notification received in configured channel" width="700">

> Save each screenshot inside a `screenshots/` folder in the repo root using the filenames above (or update the paths in this file to match your own naming).

---

## Key Learnings

- How Prometheus's pull-based scraping model works alongside exporters like Node Exporter
- Writing PromQL from scratch to calculate percentages and rates from raw counter metrics
- The difference between a Prometheus alert rule and a Grafana-native alert rule
- How Grafana alert states move through **Normal → Pending → Firing** based on the `for` duration
- Practical debugging of "target down" issues (firewall/port access, service status, scrape config)

---

## Connect

If you're on a similar DevOps learning journey, feel free to connect or follow along:

- **LinkedIn:** [linkedin.com/in/sinshac](https://linkedin.com/in/sinshac/)
- **GitHub:** [github.com/sinsha-c](https://github.com/sinsha-c)

#AWSDevOpsRestartJourney #DevOps #Prometheus #Grafana #Monitoring #Linux