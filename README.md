# Server Monitoring Templates

Production-ready monitoring templates for VPS and cloud servers.

## Features

- **Prometheus** - Metrics collection and alerting
- **Grafana** - Beautiful visualization dashboards
- **Node Exporter** - System metrics exporter
- **AlertManager** - Alert routing and notification

## Quick Start

```bash
# Clone and deploy
git clone https://github.com/clashforwindows-net/server-monitoring.git
cd server-monitoring
docker-compose up -d
```

## Dashboards Included

| Dashboard | Description |
|-----------|-------------|
| Node Overview | CPU, Memory, Disk, Network |
| Docker Stats | Container resource usage |
| Nginx Metrics | Request rate, latency, errors |
| MySQL Performance | Query stats, connections |

## Alert Rules

- CPU usage > 80% for 5min
- Memory usage > 90%
- Disk usage > 85%
- Service down detection

## Configuration

Edit `prometheus.yml` to add your targets:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

## Sponsor

[VPS推荐](https://vpsvip.net) - 优质VPS评测与推荐

---
*Last updated: 2026-04-27*
