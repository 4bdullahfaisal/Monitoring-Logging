# Prometheus & Grafana — Monitoring Stack

---

## What is Prometheus?

Prometheus is an open-source monitoring and alerting toolkit.

**Analogy:** Prometheus is like a security camera that constantly records what's happening on your servers.

---

## What is Grafana?

Grafana is an open-source analytics and visualization platform.

**Analogy:** Grafana is like a dashboard that shows you all the security camera footage in one place.

---

## Why Prometheus + Grafana?

| Problem | Solution |
|---------|----------|
| Server goes down | Get alert immediately |
| Slow response times | See latency trends |
| Disk filling up | Predict when it will be full |
| CPU spiking | Identify which process is causing it |
| Need to see historical data | Dashboard with graphs |

---

## Prometheus Architecture

| Component | What it does |
|-----------|--------------|
| **Prometheus Server** | Collects and stores metrics |
| **Exporters** | Expose metrics from applications/services |
| **Alertmanager** | Handles alerts (email, Slack, etc.) |
| **Node Exporter** | Exports system metrics (CPU, memory, disk) |
| **Pushgateway** | For short-lived jobs |
| **Grafana** | Visualizes metrics |

---

## Key Prometheus Concepts

| Term | Meaning |
|------|---------|
| **Metric** | A measurement (e.g., CPU usage) |
| **Time Series** | Metric over time with timestamps |
| **Label** | Key-value pair for filtering (e.g., `instance="web1"`) |
| **Scrape** | Pulling metrics from a target |
| **Target** | Where Prometheus collects metrics from |
| **Expression** | Query language for metric manipulation |

---

## Installing Prometheus (Docker)

### docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  prometheus_data:
  grafana_data:
```

---

## Prometheus Configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

---

## Starting the Stack

```bash
# Create directory
mkdir prometheus-grafana
cd prometheus-grafana

# Create prometheus.yml
nano prometheus.yml
# Paste the configuration above

# Create docker-compose.yml
nano docker-compose.yml
# Paste the compose file above

# Start all services
docker compose up -d

# Check status
docker compose ps
```

---

## Access the Services

| Service | URL | Default Login |
|---------|-----|---------------|
| Prometheus | http://localhost:9090 | None |
| Node Exporter | http://localhost:9100/metrics | None |
| Grafana | http://localhost:3000 | admin / admin |

---

## Prometheus Query Examples

```promql
# CPU usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# Network traffic in/out
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# All targets
up

# All metrics
count({__name__=~".+"})
```

---

## Adding Grafana Data Source

1. Login to Grafana (http://localhost:3000)
2. Click **Configuration** → **Data Sources**
3. Click **Add data source**
4. Select **Prometheus**
5. Set URL: `http://prometheus:9090`
6. Click **Save & Test**

---

## Importing a Dashboard

1. Click **+** → **Import**
2. Enter Dashboard ID: `1860` (Node Exporter Full)
3. Click **Load**
4. Select Prometheus data source
5. Click **Import**

### Popular Grafana Dashboards

| Dashboard | ID | Purpose |
|-----------|-----|---------|
| Node Exporter Full | 1860 | System metrics (CPU, memory, disk) |
| Docker Monitoring | 193 | Container metrics |
| Kubernetes Cluster | 6417 | K8s monitoring |
| Prometheus 2.0 Stats | 3662 | Prometheus itself |

---

## Alertmanager Configuration

### alertmanager.yml

```yaml
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'email'

receivers:
  - name: 'email'
    email_configs:
      - to: 'youremail@example.com'
        from: 'alert@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email'
        auth_password: 'your-password'
```

### prometheus.yml (add alerting)

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts.yml"
```

### alerts.yml

```yaml
groups:
  - name: instance
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "Instance {{ $labels.instance }} has been down for more than 1 minute."

      - alert: HighCPU
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes."

      - alert: HighMemory
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 80% for 5 minutes."

      - alert: DiskFull
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk is filling up on {{ $labels.instance }}"
          description: "Disk usage is above 85% for 5 minutes."
```

---

## Exposing Your Own Application Metrics

### Python example with prometheus_client

```python
# app.py
from prometheus_client import start_http_server, Counter, Histogram
import random
import time

# Create metrics
REQUESTS = Counter('http_requests_total', 'Total HTTP requests')
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')

@REQUEST_DURATION.time()
def handle_request():
    REQUESTS.inc()
    time.sleep(random.uniform(0.1, 0.5))
    return "OK"

if __name__ == '__main__':
    start_http_server(8000)
    print("Metrics server running on http://localhost:8000")
    while True:
        handle_request()
        time.sleep(1)
```

```bash
pip install prometheus-client
python app.py
```

Then add to prometheus.yml:
```yaml
- job_name: 'myapp'
  static_configs:
    - targets: ['host.docker.internal:8000']
```

---

## Prometheus vs Other Monitoring Tools

| Feature | Prometheus | Nagios | Zabbix | Datadog |
|---------|------------|--------|--------|---------|
| Open Source | ✅ | ✅ | ✅ | ❌ (Commercial) |
| Pull vs Push | Pull | Both | Push | Push |
| Service Discovery | ✅ | ❌ | ❌ | ✅ |
| Alerting | ✅ | ✅ | ✅ | ✅ |
| Cost | Free | Free | Free | Paid |
| Use Case | Cloud-native | Legacy | Legacy | Enterprise |

---

## Quick Commands Summary

```bash
# Docker Compose commands
docker compose up -d          # Start all services
docker compose down           # Stop and remove
docker compose ps             # Check status
docker compose logs prometheus # View logs

# Prometheus
http://localhost:9090         # Web UI
http://localhost:9090/graph   # Query UI
http://localhost:9090/targets # Targets status

# Node Exporter
http://localhost:9100/metrics # Raw metrics

# Grafana
http://localhost:3000         # Dashboard (admin/admin)

# Alertmanager
http://localhost:9093         # Alerts UI
```

---

## Quick Interview Answers

**Q: What is Prometheus?**
> "Prometheus is an open-source monitoring and alerting toolkit that collects metrics from targets via HTTP pull."

**Q: What is Grafana?**
> "Grafana is an open-source analytics and visualization platform that creates dashboards from data sources like Prometheus."

**Q: How does Prometheus collect metrics?**
> "Prometheus pulls (scrapes) metrics from targets at configured intervals over HTTP."

**Q: What is a Node Exporter?**
> "Node Exporter exports system metrics like CPU, memory, disk, and network from a Linux machine."

**Q: How do you create a Grafana dashboard?**
> "You add a data source (Prometheus), import or create panels with PromQL queries, and arrange them into a dashboard."

**Q: What is PromQL?**
> "Prometheus Query Language — used to query and aggregate time series data in Prometheus."

---


---

**End of Prometheus & Grafana Basics**
