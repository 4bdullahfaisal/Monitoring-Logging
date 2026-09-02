# Monitoring Stack with Prometheus, Grafana, AlertManager, and Loki

A comprehensive monitoring solution for DevOps learning and production-ready infrastructure monitoring. This stack includes metrics collection, visualization, alerting, and log aggregation.

![Monitoring Stack](https://img.shields.io/badge/Monitoring_Stack-Active-success?style=flat&logo=grafana)
![Prometheus](https://img.shields.io/badge/Prometheus-v2.45+-brightgreen?style=flat&logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-v10.0+-informational?style=flat&logo=grafana)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-Supported-blue?style=flat&logo=docker)

## Overview

This project provides a complete monitoring ecosystem using Docker Compose, featuring:

- **Prometheus** - Metrics collection and storage
- **Grafana** - Data visualization and dashboards
- **AlertManager** - Alert handling and notification management
- **Node Exporter** - System metrics collection
- **Loki** - Log aggregation system
- **Promtail** - Log collection and shipping

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network: Monitoring               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌─────────────────┐      │
│  │  Grafana │◄───│Prometheus│    │  AlertManager   │      │
│  │  :3001   │    │  :9090   │    │    :9093        │      │
│  └──────────┘    └────┬─────┘    └────────┬────────┘      │
│                       │                   │                │
│                       ▼                   ▼                │
│  ┌──────────┐    ┌──────────┐    ┌─────────────────┐      │
│  │   Loki   │◄───│ Promtail │    │  Node Exporter  │      │
│  │  :3100   │    │  :9080   │    │    :9100        │      │
│  └──────────┘    └──────────┘    └─────────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Docker and Docker Compose installed
- Git (for cloning the repository)
- Basic understanding of monitoring concepts

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/monitoring-stack.git
cd monitoring-stack
```

### 2. Configure AlertManager

Edit `alertmanager/alertmanager.yml` and replace `SLACK_URL_WEBHOOK` with your actual Slack webhook URL:

```yaml
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: 'alerts'
        username: 'Prometheus Alert'
        icon_emoji: ':warning:'
```

### 3. Start the Stack

```bash
docker-compose up -d
```

### 4. Verify Services

Check if all services are running:

```bash
docker-compose ps
```

Access the services:

| Service | URL | Credentials |
|---------|-----|-------------|
| Prometheus | http://localhost:9090 | - |
| AlertManager | http://localhost:9093 | - |
| Grafana | http://localhost:3001 | admin/admin |
| Loki | http://localhost:3100 | - |
| Node Exporter | http://localhost:9100 | - |

## Configuration Details

### Prometheus Configuration

`prometheus/prometheus.yml`:
- **Global Settings**: Scrape interval 15s, evaluation interval 15s
- **Alerting**: Connected to AlertManager at `alertmanager:9093`
- **Rule Files**: Loads alerts from `/etc/prometheus/alerts.yml`
- **Scrape Configs**: 
  - Prometheus self-monitoring
  - Node Exporter for system metrics

### AlertManager Configuration

`alertmanager/alertmanager.yml`:
- **Grouping**: Alerts grouped by `alertname`
- **Timings**: 
  - `group_wait`: 10s (wait before sending first alert)
  - `group_interval`: 10s (interval between sending alerts in a group)
  - `repeat_interval`: 1h (resend interval for unresolved alerts)
- **Receiver**: Slack notifications with custom message format

### Alert Rules

`alerts.yml` contains a test alert configuration:

```yaml
- alert: TestAlert
  expr: vector(1) == 1
  labels:
    severity: info
  annotations:
    summary: "✅ Test Alert - Slack is working!"
    description: "This is a test alert to verify Slack integration."
```

### Loki Configuration

`loki/loki-config.yml`:
- Uses TSDB (Time Series Database) for storage
- Filesystem storage for chunks and rules
- Ingestion limits: 10MB/s, burst 20MB

### Promtail Configuration

`promtail/promtail-config.yml`:
- Collects system logs from `/var/log/*log`
- Docker container logs via Docker socket
- Sends logs to Loki at `http://loki:3100`

## Usage Examples

### Test Alert Notification

To test the Slack integration, the test alert triggers automatically. Check your Slack channel for the notification:

```json
*Alert:* ✅ Test Alert - Slack is working!
*Description:* This is a test alert to verify Slack integration.
*Severity:* info
*Instance:* localhost:9090
```

### Querying Prometheus

Access Prometheus UI at `http://localhost:9090` and try these queries:

```promql
# CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# System uptime
time() - node_boot_time_seconds
```

### Accessing Logs in Loki

Use LogQL queries in Grafana:

```logql
{job="varlogs"} |= "error"
{container="prometheus"} |= "alert"
```

## Grafana Dashboards

### Add Prometheus Data Source

1. Go to Configuration → Data Sources
2. Add Prometheus
3. URL: `http://prometheus:9090`
4. Click "Save & Test"

### Add Loki Data Source

1. Go to Configuration → Data Sources
2. Add Loki
3. URL: `http://loki:3100`
4. Click "Save & Test"

### Recommended Dashboards

Import these dashboard IDs from Grafana.com:

- **Node Exporter Full**: 1860
- **Prometheus 2.0 Overview**: 3662
- **Docker Monitoring**: 893
- **Loki Logs**: 12019

## Project Structure

```
monitoring-stack/
├── docker-compose.yml
├── alerts.yml
├── prometheus/
│   └── prometheus.yml
├── alertmanager/
│   └── alertmanager.yml
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

## Security Considerations

### Production Recommendations

1. **Enable Authentication**: Add basic auth or OAuth for Grafana
2. **Use Environment Variables**: For sensitive data like Slack URLs
3. **Network Isolation**: Use separate networks for internal services
4. **TLS/SSL**: Enable HTTPS for all exposed endpoints
5. **Regular Updates**: Keep images up-to-date with security patches

### Environment Variables Example

Create `.env` file:

```bash
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
GF_SECURITY_ADMIN_PASSWORD=your_secure_password
```

Update `alertmanager.yml`:

```yaml
api_url: '${SLACK_WEBHOOK_URL}'
```

## Troubleshooting

### Common Issues

**1. Prometheus can't scrape targets**
```bash
# Check service connectivity
docker-compose exec prometheus ping node-exporter
```

**2. AlertManager not sending notifications**
```bash
# Check AlertManager logs
docker-compose logs alertmanager

# Verify Slack webhook URL
curl -X POST -H 'Content-type: application/json' --data '{"text":"Test"}' YOUR_WEBHOOK_URL
```

**3. Loki not receiving logs**
```bash
# Check Promtail logs
docker-compose logs promtail

# Verify Loki endpoints
curl http://localhost:3100/ready
```

**4. Docker container logs not showing**
```bash
# Check if docker_sd_configs is working
docker-compose exec promtail cat /tmp/positions.yaml
```

### Reset Stack

```bash
# Stop and remove containers
docker-compose down

# Remove volumes
docker-compose down -v

# Start fresh
docker-compose up -d
```

## Scaling Considerations

### Horizontal Scaling

- Use external storage (S3, GCS, etc.) for Loki
- Configure Prometheus with remote storage
- Use load balancers for AlertManager cluster

### Performance Tuning

- Adjust scrape intervals based on metric importance
- Use recording rules for expensive queries
- Configure retention policies in Loki

## Health Checks

### Endpoints for Health Monitoring

| Service | Health Check Endpoint |
|---------|----------------------|
| Prometheus | `http://localhost:9090/-/ready` |
| AlertManager | `http://localhost:9093/-/ready` |
| Loki | `http://localhost:3100/ready` |
| Grafana | `http://localhost:3001/api/health` |

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## Learning Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [AlertManager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Node Exporter Documentation](https://prometheus.io/docs/guides/node-exporter/)

## License

This project is licensed under the MIT License - see the LICENSE file for details.
