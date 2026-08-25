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
| **Prometheus** | Metrics collection and alert rules |
| **Exporters** | Expose metrics from applications/services |
| **Alertmanager** | Handles alerts (email, Slack, etc.) |
| **Node Exporter** | Exports system metrics (CPU, memory, disk) |
| **Pushgateway** | For short-lived jobs |
| **Grafana** | Visualizes metrics |
| **Loki** | Log aggregation  |
| **Promtail** | Log collection from system and Docker |

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

## Project Structure

```
monitoring-stack/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── alertmanager/
│   └── alertmanager.yml
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
└── alerts.yml
```

---

## File 1: `docker-compose.yml`

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring

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
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"  # Changed from 3000 to 3001
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command:
      - '-config.file=/etc/loki/loki-config.yml'
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command:
      - '-config.file=/etc/promtail/promtail-config.yml'
    networks:
      - monitoring
    depends_on:
      - loki

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
  loki_data:

networks:
  monitoring:
    driver: bridge
```

---

## File 2: `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "/etc/prometheus/alerts.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

---

## File 3: `alertmanager/alertmanager.yml`

```yaml
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/your_token'
        channel: '#alerts'
        username: 'Prometheus Alert'
        icon_emoji: ':warning:'
        text: |-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
            *Instance:* {{ .Labels.instance }}
            *Value:* {{ .Annotations.value }}
          {{ end }}

web:
  external_url: 'http://localhost:9093'
```

---

## File 4: `alerts.yml`

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
          summary: "🚨 Instance {{ $labels.instance }} down"
          description: "Instance {{ $labels.instance }} has been down for more than 1 minute."

      - alert: HighCPU
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "⚠️ High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes."

      - alert: HighMemory
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "⚠️ High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 80% for 5 minutes."

      - alert: TestAlert
        expr: 1 == 1
        for: 0s
        labels:
          severity: info
        annotations:
          summary: "✅ Test Alert - Slack is working!"
          description: "This is a test alert to verify Slack integration is working correctly."
          value: "Test notification sent at {{ now.Format \"15:04:05\" }}"
```

---

## File 5: `loki/loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks

limits_config:
  allow_structured_metadata: true

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

compactor:
  working_directory: /loki/compactor
```

---

## File 6: `promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'logstream'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'
```

---

## Deployment Steps

### 1. Create all directories and files:

```bash
# Create project directory
mkdir monitoring-stack
cd monitoring-stack

# Create directories
mkdir -p prometheus alertmanager loki promtail

# Create all files (copy content from above)
nano docker-compose.yml
nano prometheus/prometheus.yml
nano alertmanager/alertmanager.yml
nano loki/loki-config.yml
nano promtail/promtail-config.yml
nano alerts.yml
```

### 2. Start the stack:

```bash
docker compose up -d
```

### 3. Check all services:

```bash
docker compose ps
```

**Expected output:**
```
NAME           STATUS
prometheus     Up
alertmanager   Up
node-exporter  Up
grafana        Up
loki           Up
promtail       Up
```

---

## Access URLs

| Service | URL |
|---------|-----|
| **Prometheus** | http://localhost:9090 |
| **Alertmanager** | http://localhost:9093 |
| **Node Exporter** | http://localhost:9100/metrics |
| **Grafana** | http://localhost:3001 |
| **Loki** | http://localhost:3100 |

---

## Grafana Login

```
URL: http://localhost:3001
Username: admin
Password: admin
```

---

## 📊 Add Data Sources in Grafana

### 1. Add Prometheus:
1. Login to Grafana (http://localhost:3001)
2. Gear icon → **Data Sources** → **Add data source**
3. Select **Prometheus**
4. URL: `http://prometheus:9090`
5. Click **Save & Test**

### 2. Add Loki:
1. **Add data source**
2. Select **Loki**
3. URL: `http://loki:3100`
4. Click **Save & Test**

---

## 📱 Test Slack Integration

### The test alert will fire immediately:

1. Go to **Alertmanager**: http://localhost:9093/#/alerts
2. You should see the **"TestAlert"** in "Firing" state
3. Check your Slack **#alerts** channel - you should receive:

```
Prometheus Alert
:warning: 
Alert: ✅ Test Alert - Slack is working!
Description: This is a test alert to verify Slack integration is working correctly.
Severity: info
Instance: prometheus
Value: Test notification sent at 14:32:15
```
---

### Popular Grafana Dashboards

| Dashboard | ID | Purpose |
|-----------|-----|---------|
| Node Exporter Full | 1860 | System metrics (CPU, memory, disk) |
| Docker Monitoring | 193 | Container metrics |
| Kubernetes Cluster | 6417 | K8s monitoring |
| Prometheus 2.0 Stats | 3662 | Prometheus itself |

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
# Check all containers are running
docker compose ps

# Check Prometheus alerts
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'

# Check Alertmanager alerts
curl http://localhost:9093/api/v2/alerts | jq '.[] | {name: .labels.alertname, status: .status.state}'

# Check Loki is working
curl http://localhost:3100/loki/api/v1/labels

# View Alertmanager logs
docker logs alertmanager

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
