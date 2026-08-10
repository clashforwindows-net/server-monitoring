# Server Monitoring Templates — 生产级监控栈完整方案

> Prometheus + Grafana + Alertmanager 的**开箱即用模板**，附带告警规则设计、SLO 定义、容量规划方法与运维实践。
> 目标：**30 分钟搭起监控，30 天后依然有人在看。**

![Prometheus](https://img.shields.io/badge/Prometheus-2.5x-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-11.x-F46800?logo=grafana&logoColor=white)
![Docker](https://img.shields.io/badge/Docker%20Compose-v2-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Table of Contents

- [1. 为什么大部分监控最后没人看](#1-为什么大部分监控最后没人看)
- [2. 架构设计](#2-架构设计)
- [3. 快速部署](#3-快速部署)
- [4. Prometheus 配置详解](#4-prometheus-配置详解)
- [5. Exporter 选型与部署](#5-exporter-选型与部署)
- [6. 告警规则库](#6-告警规则库)
- [7. Alertmanager 路由与静默](#7-alertmanager-路由与静默)
- [8. Grafana 仪表盘](#8-grafana-仪表盘)
- [9. PromQL 实战速查](#9-promql-实战速查)
- [10. SLI / SLO / 错误预算](#10-sli--slo--错误预算)
- [11. 日志侧：Loki 集成](#11-日志侧loki-集成)
- [12. 存储、保留期与容量规划](#12-存储保留期与容量规划)
- [13. 高可用与远程写](#13-高可用与远程写)
- [14. 安全加固](#14-安全加固)
- [15. 性能基准与调优](#15-性能基准与调优)
- [16. 故障排查](#16-故障排查)
- [17. FAQ](#17-faq)
- [18. 相关资源](#18-相关资源)

---

## 1. 为什么大部分监控最后没人看

见过太多这样的监控系统：

| 反模式 | 后果 |
|--------|------|
| 装了 500 个指标，一个都没配告警 | 出事了还是靠用户报障 |
| 告警阈值拍脑袋定（CPU>80% 就报） | 每天几十条误报，最后全员静音 |
| 告警没有可执行动作 | 收到"MemoryHigh"不知道该干嘛 |
| 仪表盘 30 个面板挤一屏 | 打开就晕，没人愿意看 |
| 只监控机器，不监控业务 | 机器全绿，用户下不了单 |

本模板的设计原则：

1. **告警必须可执行** —— 每条告警都带 `summary`、`description`、`runbook_url`
2. **分级而非平铺** —— P1 电话/P2 IM/P3 只记录，不同级别不同通道
3. **先做黑盒再做白盒** —— 用户视角（能不能访问）优先于机器视角（CPU 多少）
4. **面板服务于问题** —— 每个仪表盘回答一个明确问题，不做"数据大杂烩"

---

## 2. 架构设计

```
                ┌──────────────────────────────────────────┐
                │              被监控目标                    │
                │  node_exporter  cadvisor  nginx_exporter  │
                │  mysqld_exporter  blackbox  自定义 /metrics│
                └────────────────┬─────────────────────────┘
                                 │ pull (HTTP /metrics)
                                 ▼
                       ┌──────────────────┐
                       │    Prometheus    │◄── 规则评估 (recording + alerting)
                       │  TSDB 本地存储    │
                       └────┬────────┬────┘
                            │        │ 告警
                   查询     │        ▼
                            │  ┌──────────────┐
                            │  │ Alertmanager │──► 分组/抑制/静默/去重
                            │  └──────┬───────┘
                            │         ├──► Webhook (IM)
                            │         ├──► Email
                            │         └──► PagerDuty / 电话
                            ▼
                     ┌─────────────┐
                     │   Grafana   │──► 仪表盘 / 探索 / 告警面板
                     └─────────────┘
```

### 组件职责

| 组件 | 职责 | 端口 |
|------|------|:----:|
| Prometheus | 拉取、存储、规则评估 | 9090 |
| Alertmanager | 告警路由、分组、抑制、静默 | 9093 |
| Grafana | 可视化、探索、看板 | 3000 |
| node_exporter | 主机指标 | 9100 |
| cAdvisor | 容器指标 | 8080 |
| blackbox_exporter | 黑盒探测（HTTP/TCP/ICMP/DNS） | 9115 |
| nginx_exporter | Nginx 指标 | 9113 |
| mysqld_exporter | MySQL 指标 | 9104 |
| Loki + Promtail | 日志聚合 | 3100 |

### Pull vs Push

Prometheus 采用 **pull 模型**：由 Prometheus 主动去拉目标的 `/metrics`。

**好处**：目标挂了自然产生 `up == 0`（天然的存活检测）；配置集中在 Prometheus 侧；容易做本地调试（浏览器直接打开 `/metrics`）。

**例外场景**（短任务/批处理）用 **Pushgateway**：

```bash
echo "backup_last_success_timestamp $(date +%s)" | \
  curl --data-binary @- http://pushgateway:9091/metrics/job/mysql_backup/instance/db01
```

> ⚠️ Pushgateway 只用于**生命周期短于抓取间隔的任务**，不要拿它当通用推送网关，否则会丢失实例存活语义。

---

## 3. 快速部署

### 3.1 目录结构

```
server-monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   ├── targets/
│   │   ├── nodes.yml
│   │   └── blackbox.yml
│   └── rules/
│       ├── node.rules.yml
│       ├── container.rules.yml
│       ├── blackbox.rules.yml
│       ├── mysql.rules.yml
│       └── recording.rules.yml
├── alertmanager/
│   ├── alertmanager.yml
│   └── templates/
│       └── custom.tmpl
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/datasource.yml
│   │   └── dashboards/dashboard.yml
│   └── dashboards/
│       ├── node-overview.json
│       ├── docker-stats.json
│       ├── nginx.json
│       └── slo.json
├── blackbox/
│   └── blackbox.yml
└── loki/
    ├── loki-config.yml
    └── promtail-config.yml
```

### 3.2 docker-compose.yml

```yaml
name: monitoring

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "20m"
    max-file: "3"

services:
  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: prometheus
    restart: unless-stopped
    user: "65534:65534"
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=40GB'
      - '--web.enable-lifecycle'
      - '--web.external-url=https://prom.example.com'
    ports:
      - "127.0.0.1:9090:9090"
    logging: *default-logging

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager:/etc/alertmanager:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--data.retention=120h'
    ports:
      - "127.0.0.1:9093:9093"
    logging: *default-logging

  grafana:
    image: grafana/grafana:11.1.0
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:?set in .env}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: https://grafana.example.com
      GF_ANALYTICS_REPORTING_ENABLED: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "127.0.0.1:3000:3000"
    depends_on: [prometheus]
    logging: *default-logging

  node-exporter:
    image: prom/node-exporter:v1.8.1
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    network_mode: host
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'
      - '--collector.systemd'
      - '--collector.processes'
      - '--collector.textfile.directory=/host/var/lib/node_exporter'
      - '--no-collector.hwmon'
    logging: *default-logging

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "127.0.0.1:8080:8080"
    logging: *default-logging

  blackbox:
    image: prom/blackbox-exporter:v0.25.0
    container_name: blackbox
    restart: unless-stopped
    volumes:
      - ./blackbox:/etc/blackbox_exporter:ro
    ports:
      - "127.0.0.1:9115:9115"
    logging: *default-logging

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
```

### 3.3 启动

```bash
git clone https://github.com/clashforwindows-net/server-monitoring.git
cd server-monitoring
cp .env.example .env && vim .env      # 设置 GRAFANA_PASSWORD
docker compose up -d
docker compose ps

# 校验配置（改配置后必做）
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose exec prometheus promtool check rules /etc/prometheus/rules/*.yml

# 热重载，不用重启容器
curl -X POST http://127.0.0.1:9090/-/reload
```

> ⚠️ 注意 compose 里所有端口都绑到了 `127.0.0.1`。**Prometheus 和 cAdvisor 默认没有任何认证**，直接暴露公网等于把服务器内部结构全部公开。访问请走 Nginx 反代 + Basic Auth（见第 14 章）。

---

## 4. Prometheus 配置详解

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    cluster: prod
    region: hk

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 用文件服务发现，加机器不用改主配置也不用重启
  - job_name: 'node'
    file_sd_configs:
      - files: ['/etc/prometheus/targets/nodes.yml']
        refresh_interval: 30s
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance
        replacement: '$1'

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metric_relabel_configs:
      # cAdvisor 指标量极大，丢弃用不上的以省存储
      - source_labels: [__name__]
        regex: 'container_(tasks_state|memory_failures_total|blkio_device_usage_total)'
        action: drop
      - source_labels: [id]
        regex: '/system.slice/.*'
        action: drop

  # 黑盒探测：用户视角的可用性
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    file_sd_configs:
      - files: ['/etc/prometheus/targets/blackbox.yml']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

`prometheus/targets/nodes.yml`：

```yaml
- targets: ['10.0.1.11:9100', '10.0.1.12:9100']
  labels: {env: prod, role: web}
- targets: ['10.0.2.21:9100']
  labels: {env: prod, role: db}
- targets: ['10.0.3.31:9100']
  labels: {env: staging, role: web}
```

**关键点**：用 `file_sd_configs` 而不是 `static_configs`，加机器只需改这个文件，Prometheus 每 30 秒自动重载，**不用重启也不用 reload**。

### 4.1 Recording Rules（预计算）

高频查询的复杂表达式提前算好，仪表盘打开速度差别巨大。

```yaml
# rules/recording.rules.yml
groups:
  - name: node_recording
    interval: 30s
    rules:
      - record: instance:node_cpu_utilization:rate5m
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

      - record: instance:node_filesystem_usage:ratio
        expr: |
          1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
               / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})

      - record: instance:node_network_receive_bytes:rate5m
        expr: sum by (instance) (rate(node_network_receive_bytes_total{device!~"lo|veth.*|docker.*"}[5m]))

      - record: instance:node_load_per_core:ratio
        expr: node_load5 / on(instance) count by (instance)(node_cpu_seconds_total{mode="idle"})
```

**命名规范**：`level:metric:operation`，例如 `instance:node_cpu_utilization:rate5m`。

---

## 5. Exporter 选型与部署

| 监控对象 | Exporter | 关键指标 |
|----------|----------|----------|
| 主机 | node_exporter | CPU/内存/磁盘/网络/文件系统 |
| Docker | cAdvisor | 容器 CPU/内存/网络/重启 |
| Nginx | nginx-prometheus-exporter | 连接数、请求速率 |
| MySQL | mysqld_exporter | QPS、慢查询、连接数、复制延迟 |
| Redis | redis_exporter | 命中率、内存、连接、慢日志 |
| PostgreSQL | postgres_exporter | 事务、锁、膨胀 |
| 证书/域名 | blackbox_exporter | 证书剩余天数、探测状态 |
| 任意脚本 | node_exporter textfile | 自定义业务指标 |

### 5.1 自定义指标（textfile collector）

不想写 exporter？把指标写成文件就行：

```bash
#!/usr/bin/env bash
# /usr/local/bin/custom-metrics.sh  → cron 每分钟跑
OUT=/var/lib/node_exporter/custom.prom
TMP="${OUT}.$$"

{
  echo '# HELP app_backup_last_success_timestamp 上次备份成功时间'
  echo '# TYPE app_backup_last_success_timestamp gauge'
  echo "app_backup_last_success_timestamp $(stat -c %Y /data/backup/latest.sql.gz 2>/dev/null || echo 0)"

  echo '# HELP app_queue_pending 待处理队列长度'
  echo '# TYPE app_queue_pending gauge'
  echo "app_queue_pending $(redis-cli LLEN task_queue 2>/dev/null || echo 0)"

  echo '# HELP ssl_cert_expiry_days 证书剩余天数'
  echo '# TYPE ssl_cert_expiry_days gauge'
  for d in example.com api.example.com; do
    exp=$(echo | openssl s_client -servername "$d" -connect "$d:443" 2>/dev/null \
          | openssl x509 -noout -enddate | cut -d= -f2)
    days=$(( ( $(date -d "$exp" +%s) - $(date +%s) ) / 86400 ))
    echo "ssl_cert_expiry_days{domain=\"$d\"} $days"
  done
} > "$TMP"

mv "$TMP" "$OUT"     # 原子替换，避免读到写了一半的文件
```

### 5.2 blackbox 配置

```yaml
# blackbox/blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 8s
    http:
      valid_status_codes: [200, 201, 204, 301, 302]
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: ip4
      fail_if_body_not_matches_regexp: []

  http_post_api:
    prober: http
    timeout: 10s
    http:
      method: POST
      headers: {Content-Type: application/json}
      body: '{"health":"check"}'

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp_ping:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  dns_resolve:
    prober: dns
    dns:
      query_name: example.com
      query_type: A
```

---

## 6. 告警规则库

### 6.1 主机告警

```yaml
# rules/node.rules.yml
groups:
  - name: node_alerts
    rules:
      - alert: InstanceDown
        expr: up{job="node"} == 0
        for: 2m
        labels: {severity: critical, team: infra}
        annotations:
          summary: "实例 {{ $labels.instance }} 失联"
          description: "已持续 2 分钟无法抓取指标。先确认是网络问题还是机器宕机。"
          runbook_url: "https://wiki.example.com/runbook/instance-down"

      - alert: HighCPUUsage
        expr: instance:node_cpu_utilization:rate5m > 85
        for: 10m
        labels: {severity: warning, team: infra}
        annotations:
          summary: "{{ $labels.instance }} CPU 使用率 {{ $value | printf \"%.1f\" }}%"
          description: "持续 10 分钟高于 85%。执行 `pidstat 1 5` 定位进程。"

      - alert: HighCPUSteal
        expr: rate(node_cpu_seconds_total{mode="steal"}[5m]) * 100 > 10
        for: 15m
        labels: {severity: warning, team: infra}
        annotations:
          summary: "{{ $labels.instance }} CPU steal {{ $value | printf \"%.1f\" }}%"
          description: "宿主机资源被抢占，优化自身代码无效，考虑更换服务商或迁移实例。"

      - alert: HighMemoryUsage
        expr: instance:node_memory_utilization:ratio > 0.90
        for: 10m
        labels: {severity: warning, team: infra}
        annotations:
          summary: "{{ $labels.instance }} 内存使用率 {{ $value | humanizePercentage }}"

      - alert: OOMKillDetected
        expr: increase(node_vmstat_oom_kill[10m]) > 0
        labels: {severity: critical, team: infra}
        annotations:
          summary: "{{ $labels.instance }} 发生 OOM Kill"
          description: "执行 `dmesg -T | grep -i oom` 查看被杀进程。"

      - alert: DiskSpaceWarning
        expr: instance:node_filesystem_usage:ratio > 0.85
        for: 15m
        labels: {severity: warning, team: infra}
        annotations:
          summary: "{{ $labels.instance }} {{ $labels.mountpoint }} 已用 {{ $value | humanizePercentage }}"

      # 比"当前用了多少"更有价值：预测什么时候满
      - alert: DiskWillFillIn24h
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24*3600) < 0
          and instance:node_filesystem_usage:ratio > 0.70
        for: 30m
        labels: {severity: critical, team: infra}
        annotations:
          summary: "{{ $labels.instance }} {{ $labels.mountpoint }} 预计 24h 内写满"
          description: "基于近 6 小时增长趋势线性外推。现在处理还来得及。"

      - alert: DiskInodeHigh
        expr: |
          1 - (node_filesystem_files_free / node_filesystem_files) > 0.85
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} inode 使用率过高"
          description: "空间够但 inode 快满了，会报 No space left on device。"

      - alert: HighDiskIOWait
        expr: rate(node_cpu_seconds_total{mode="iowait"}[5m]) * 100 > 25
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} IO wait {{ $value | printf \"%.1f\" }}%"
          description: "用 `iotop -oPa` 找出 IO 大户。"

      - alert: HighLoadPerCore
        expr: instance:node_load_per_core:ratio > 2
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 单核负载 {{ $value | printf \"%.2f\" }}"

      - alert: UnexpectedReboot
        expr: node_time_seconds - node_boot_time_seconds < 300
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 在 5 分钟内重启过"
          description: "检查 `journalctl -b -1 -p err`。"

      - alert: ClockSkew
        expr: abs(node_timex_offset_seconds) > 0.5
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 时钟偏移 {{ $value }}s"
          description: "会导致 TLS 校验、日志时序、分布式锁异常。"
```

### 6.2 容器告警

```yaml
# rules/container.rules.yml
groups:
  - name: container_alerts
    rules:
      - alert: ContainerRestartLoop
        expr: increase(container_start_time_seconds{name!=""}[15m]) > 3
        labels: {severity: critical}
        annotations:
          summary: "容器 {{ $labels.name }} 15 分钟内重启超过 3 次"
          description: "`docker logs --tail 200 {{ $labels.name }}` 查看崩溃原因。"

      - alert: ContainerMemoryNearLimit
        expr: |
          container_memory_working_set_bytes{name!=""}
          / container_spec_memory_limit_bytes{name!=""} > 0.90
          and container_spec_memory_limit_bytes{name!=""} > 0
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "容器 {{ $labels.name }} 内存接近上限，即将被 OOM Kill"

      - alert: ContainerCPUThrottled
        expr: |
          rate(container_cpu_cfs_throttled_periods_total{name!=""}[5m])
          / rate(container_cpu_cfs_periods_total{name!=""}[5m]) > 0.25
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "容器 {{ $labels.name }} CPU 被限流 {{ $value | humanizePercentage }}"
          description: "CPU limit 设置过低，请求延迟会明显上升。"
```

### 6.3 黑盒与证书告警

```yaml
# rules/blackbox.rules.yml
groups:
  - name: blackbox_alerts
    rules:
      - alert: EndpointDown
        expr: probe_success == 0
        for: 3m
        labels: {severity: critical, team: sre}
        annotations:
          summary: "{{ $labels.instance }} 探测失败"
          description: "用户视角已不可用，优先级最高。"

      - alert: SlowResponse
        expr: probe_duration_seconds > 3
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 响应耗时 {{ $value | printf \"%.2f\" }}s"

      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 14
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 证书 {{ $value | printf \"%.0f\" }} 天后过期"
          description: "检查 certbot 定时续期是否正常：`systemctl list-timers | grep certbot`"

      - alert: SSLCertExpiringCritical
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 3
        labels: {severity: critical}
        annotations:
          summary: "🔥 {{ $labels.instance }} 证书 3 天内过期，立即处理"
```

### 6.4 元监控（监控自己）

```yaml
groups:
  - name: meta_alerts
    rules:
      - alert: PrometheusConfigReloadFailed
        expr: prometheus_config_last_reload_successful == 0
        labels: {severity: critical}
        annotations:
          summary: "Prometheus 配置重载失败，当前跑的是旧配置"

      - alert: PrometheusTSDBCompactionFailing
        expr: increase(prometheus_tsdb_compactions_failed_total[3h]) > 0
        labels: {severity: warning}

      - alert: TooManyScrapeFailures
        expr: |
          count by (job) (up == 0) / count by (job) (up) > 0.3
        for: 10m
        labels: {severity: critical}
        annotations:
          summary: "job {{ $labels.job }} 超过 30% 的目标抓取失败"

      - alert: AlertmanagerNotificationFailing
        expr: rate(alertmanager_notifications_failed_total[10m]) > 0
        labels: {severity: critical}
        annotations:
          summary: "告警发不出去了 —— 这是最危险的静默故障"
```

> 💡 **元监控最容易被忽略**：如果 Alertmanager 挂了，你不会收到"Alertmanager 挂了"的告警。建议用外部的独立心跳服务（dead man's switch）：让 Prometheus 每分钟推一次心跳到外部服务，超时未收到就通知你。

```yaml
      - alert: DeadMansSwitch
        expr: vector(1)
        labels: {severity: none}
        annotations:
          summary: "心跳告警，应该一直存在。如果它消失了，说明监控系统本身挂了。"
```

---

## 7. Alertmanager 路由与静默

```yaml
global:
  resolve_timeout: 5m

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'severity']
  group_wait: 30s          # 首次等待，把同类告警攒一批
  group_interval: 5m       # 同组有新告警时的最小发送间隔
  repeat_interval: 4h      # 未恢复告警的重复提醒间隔

  routes:
    - matchers: [severity="none"]
      receiver: 'heartbeat'
      group_wait: 0s
      repeat_interval: 1m

    - matchers: [severity="critical"]
      receiver: 'oncall'
      group_wait: 10s
      repeat_interval: 1h
      continue: true       # 继续匹配后面的路由，实现多通道并发

    - matchers: [severity="warning"]
      receiver: 'im-channel'
      repeat_interval: 12h

    - matchers: [team="dba"]
      receiver: 'dba-team'

# 抑制：机器宕机时不要再报它上面所有服务的告警
inhibit_rules:
  - source_matchers: [alertname="InstanceDown"]
    target_matchers: [severity=~"warning|info"]
    equal: ['instance']

  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: ['alertname', 'instance']

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://webhook-adapter:8000/default'

  - name: 'oncall'
    webhook_configs:
      - url: 'http://webhook-adapter:8000/oncall'
        send_resolved: true

  - name: 'im-channel'
    webhook_configs:
      - url: 'http://webhook-adapter:8000/im'
        send_resolved: true

  - name: 'dba-team'
    email_configs:
      - to: 'dba@example.com'
        send_resolved: true

  - name: 'heartbeat'
    webhook_configs:
      - url: 'https://hc-ping.com/your-uuid'
```

### 7.1 静默（维护窗口）

```bash
# 命令行创建静默
amtool --alertmanager.url=http://127.0.0.1:9093 silence add \
  instance=10.0.1.11 --duration=2h --comment="计划内维护，升级内核"

amtool silence query
amtool silence expire <silence-id>
```

### 7.2 告警文案模板

```
{{ define "custom.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
{{ end }}

{{ define "custom.text" }}
{{ range .Alerts }}
🔸 *{{ .Annotations.summary }}*
   实例: {{ .Labels.instance }}
   级别: {{ .Labels.severity }}
   开始: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
   说明: {{ .Annotations.description }}
{{ if .Annotations.runbook_url }}   手册: {{ .Annotations.runbook_url }}{{ end }}
{{ end }}
{{ end }}
```

---

## 8. Grafana 仪表盘

### 8.1 自动配置数据源

```yaml
# grafana/provisioning/datasources/datasource.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

### 8.2 自动加载仪表盘

```yaml
# grafana/provisioning/dashboards/dashboard.yml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Infrastructure'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### 8.3 内置仪表盘

| 仪表盘 | 回答什么问题 | 核心面板 |
|--------|--------------|----------|
| **Node Overview** | 机器健康吗 | CPU/内存/磁盘/网络/负载/steal |
| **Docker Stats** | 容器有没有异常 | 容器资源、重启次数、限流比例 |
| **Nginx Metrics** | 流量与错误率 | QPS、状态码分布、P50/P95/P99 |
| **MySQL Performance** | 数据库健康吗 | QPS、慢查询、连接数、复制延迟 |
| **Blackbox / SLO** | 用户能用吗 | 可用率、探测延迟、证书天数、错误预算 |
| **Capacity Planning** | 什么时候需要扩容 | 磁盘增长趋势、内存趋势、预测线 |

### 8.4 推荐的社区仪表盘 ID

| ID | 名称 |
|----|------|
| 1860 | Node Exporter Full（最经典） |
| 193 | Docker monitoring with cAdvisor |
| 7587 | Blackbox Exporter |
| 7362 | MySQL Overview |
| 763 | Redis Dashboard |
| 12708 | Nginx by nginx-prometheus-exporter |

导入方式：Grafana → Dashboards → Import → 输入 ID → 选择数据源。

### 8.5 仪表盘设计原则

1. **一屏原则**：核心仪表盘不超过 12 个面板，滚动才能看完的没人看
2. **从上到下**：红绿灯状态 → 关键趋势 → 明细分解
3. **统一时间轴**：所有面板共享时间选择器，方便关联分析
4. **善用变量**：`$instance`、`$job` 下拉切换，一个看板管所有机器
5. **阈值染色**：85% 变黄、95% 变红，一眼看出问题

```
变量配置示例:
  Name:  instance
  Type:  Query
  Query: label_values(node_uname_info, instance)
  Multi-value: true
  Include All: true
```

---

## 9. PromQL 实战速查

### 基础

```promql
# 瞬时值
node_memory_MemAvailable_bytes

# 5 分钟平均增长率（counter 类型必须配 rate）
rate(node_network_receive_bytes_total[5m])

# irate：瞬时速率，适合快速变化的指标（但会有毛刺）
irate(node_cpu_seconds_total[5m])

# increase：区间内增量
increase(http_requests_total[1h])
```

### 聚合

```promql
sum by (instance) (rate(node_network_receive_bytes_total[5m]))
avg by (job) (instance:node_cpu_utilization:rate5m)
max by (instance) (instance:node_filesystem_usage:ratio)
count by (job) (up == 1)
topk(5, instance:node_cpu_utilization:rate5m)
bottomk(3, node_memory_MemAvailable_bytes)
```

### 常用表达式

```promql
# CPU 使用率
100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 磁盘使用率
(1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
   / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100

# 网络带宽 Mbps
sum by (instance)(rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[5m])) * 8 / 1000000

# HTTP 错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# P99 延迟（histogram）
histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))

# 磁盘写满预测
predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0

# 同比：本周 vs 上周
sum(rate(http_requests_total[5m]))
  / sum(rate(http_requests_total[5m] offset 7d))
```

### 常见坑

| 坑 | 说明 |
|----|------|
| counter 不加 `rate` | counter 是单调递增值，直接画出来是斜线，没有意义 |
| `rate` 窗口太小 | 窗口必须 ≥ 4 倍 scrape_interval，否则数据点不足返回空 |
| 忘记 `by` 导致标签爆炸 | `rate(x[5m])` 会返回每个时间序列，图上几百条线 |
| `irate` 用在告警里 | `irate` 只取最后两点，抖动大，告警会疯狂闪烁。**告警一律用 `rate`** |
| 除法分母为 0 | 用 `> 0` 过滤，或 `clamp_min(denom, 1)` |

---

## 10. SLI / SLO / 错误预算

### 10.1 概念

- **SLI**（指标）：可用率 = 成功请求数 / 总请求数
- **SLO**（目标）：30 天可用率 ≥ 99.9%
- **错误预算**：100% - 99.9% = 0.1% → 30 天允许 **43.2 分钟**不可用

| SLO | 月度允许不可用时间 |
|:---:|:------------------:|
| 99% | 7 小时 18 分 |
| 99.9% | 43 分 12 秒 |
| 99.95% | 21 分 36 秒 |
| 99.99% | 4 分 19 秒 |

**99.99% 对单台 VPS 来说是不现实的目标**——一次系统更新重启就用光了预算。定 SLO 要诚实。

### 10.2 用 PromQL 计算

```yaml
groups:
  - name: slo
    interval: 60s
    rules:
      - record: slo:availability:ratio_30d
        expr: avg_over_time(probe_success[30d])

      - record: slo:error_budget_remaining:ratio
        expr: |
          1 - ((1 - avg_over_time(probe_success[30d])) / (1 - 0.999))

      - alert: ErrorBudgetBurnFast
        expr: |
          (1 - avg_over_time(probe_success[1h])) > (14.4 * 0.001)
          and
          (1 - avg_over_time(probe_success[5m])) > (14.4 * 0.001)
        labels: {severity: critical}
        annotations:
          summary: "{{ $labels.instance }} 错误预算快速消耗"
          description: "按当前速率，2 天内会烧完整月预算。"
```

**多窗口燃尽率告警**（Google SRE 推荐）比固定阈值告警更少误报：短窗口确认"现在正在发生"，长窗口确认"不是一次性抖动"。

---

## 11. 日志侧：Loki 集成

```yaml
  loki:
    image: grafana/loki:3.0.0
    container_name: loki
    restart: unless-stopped
    volumes:
      - ./loki/loki-config.yml:/etc/loki/config.yml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/config.yml
    ports: ["127.0.0.1:3100:3100"]

  promtail:
    image: grafana/promtail:3.0.0
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./loki/promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
```

LogQL 示例：

```logql
{job="nginx"} |= "500"
{job="nginx"} | json | status >= 500
sum by (status) (rate({job="nginx"} | json | __error__="" [5m]))
{container="myapp"} |~ "(?i)(error|exception|panic)"
```

**Metrics + Logs 联动**：在 Grafana 里给告警面板配 Data Link 跳转到对应时间段的 Loki 查询，排查效率提升巨大。

---

## 12. 存储、保留期与容量规划

### 12.1 磁盘估算公式

```
所需磁盘 ≈ 保留天数 × 86400 × (总时间序列数 / scrape_interval) × 单样本字节数
```

单样本压缩后约 **1.5-2 字节**。

**实例：**

| 场景 | 序列数 | interval | 保留 | 磁盘 |
|------|-------:|:--------:|:----:|-----:|
| 5 台机器（仅 node_exporter） | ~5,000 | 15s | 30d | ~1.7 GB |
| 20 台 + 容器 | ~80,000 | 15s | 30d | ~28 GB |
| 50 台 + 容器 + 应用 | ~300,000 | 30s | 15d | ~52 GB |

```promql
# 查看当前序列总数
prometheus_tsdb_head_series

# 哪个 job 贡献了最多序列（找基数爆炸源头）
topk(10, count by (job) ({__name__=~".+"}))

# 哪个指标名序列最多
topk(10, count by (__name__) ({__name__=~".+"}))
```

### 12.2 基数（Cardinality）控制

**监控系统崩溃的头号原因是高基数标签。**

❌ 千万不要把这些放进 label：
- 用户 ID、订单号、请求 ID、Session ID
- 完整 URL（含查询参数）
- IP 地址（除非确实有限）
- 时间戳

✅ 正确做法：用**路由模板**代替具体路径

```
❌ http_requests_total{path="/user/12345/orders/98765"}
✅ http_requests_total{route="/user/:id/orders/:oid"}
```

用 `metric_relabel_configs` 在入库前丢弃：

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'go_gc_duration_seconds.*|promhttp_.*'
    action: drop
  - regex: 'id|container_id|pod_uid'
    action: labeldrop
```

---

## 13. 高可用与远程写

### 13.1 简单 HA：双实例并行

```yaml
# 两台 Prometheus 抓同样的 target，Alertmanager 集群自动去重
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['am1:9093', 'am2:9093']
```

```bash
# Alertmanager 集群
alertmanager --cluster.peer=am2:9094
```

Alertmanager 的 gossip 集群会对相同 fingerprint 的告警去重，**你不会收到两条一样的通知**。

### 13.2 长期存储：remote_write

```yaml
remote_write:
  - url: https://remote.example.com/api/v1/write
    basic_auth:
      username: prom
      password_file: /etc/prometheus/remote_pass
    queue_config:
      capacity: 10000
      max_shards: 50
      max_samples_per_send: 2000
      batch_send_deadline: 5s
    write_relabel_configs:
      # 只把重要指标写远端，省流量省钱
      - source_labels: [__name__]
        regex: 'instance:.*|slo:.*|up|probe_success'
        action: keep
```

可选后端：Thanos（对象存储 + 全局查询）、VictoriaMetrics（更省资源）、Mimir。

**单机场景老实说**：30 天本地保留 + 每天备份 TSDB 快照就够了，不要为了"高可用"引入三倍复杂度。

```bash
# TSDB 快照备份
curl -XPOST http://127.0.0.1:9090/api/v1/admin/tsdb/snapshot
tar czf prom-snap-$(date +%F).tgz /prometheus/snapshots/
```

---

## 14. 安全加固

### 14.1 绝不要裸奔

Prometheus、cAdvisor、node_exporter **默认零认证**。暴露公网 = 把你的内网拓扑、服务列表、版本号全部公开。

```nginx
server {
    listen 443 ssl http2;
    server_name prom.example.com;

    ssl_certificate     /etc/letsencrypt/live/prom.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/prom.example.com/privkey.pem;

    auth_basic           "Monitoring";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # 再加一层 IP 白名单
    allow 203.0.113.0/24;
    deny  all;

    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
htpasswd -c /etc/nginx/.htpasswd monitor
```

### 14.2 检查清单

```
[ ] 所有组件端口绑定 127.0.0.1 或内网 IP
[ ] 对外访问一律走 HTTPS + Basic Auth / OAuth
[ ] Grafana 关闭匿名访问与用户注册
[ ] Grafana 管理员密码从环境变量注入，不写进 compose 文件
[ ] node_exporter 用 --web.listen-address 绑内网
[ ] exporter 之间用防火墙限制，只允许 Prometheus 来源 IP
[ ] 定期升级镜像版本（订阅 CVE 通告）
[ ] 告警 webhook URL 视为密钥，不提交到 Git
```

---

## 15. 性能基准与调优

### 15.1 关键自监控指标

```promql
prometheus_tsdb_head_series                       # 活跃序列数
rate(prometheus_tsdb_head_samples_appended_total[5m])   # 每秒摄入样本数
prometheus_target_interval_length_seconds{quantile="0.99"}  # 抓取间隔实际值
rate(prometheus_rule_evaluation_duration_seconds_sum[5m])   # 规则评估耗时
prometheus_engine_query_duration_seconds{quantile="0.9"}    # 查询耗时
process_resident_memory_bytes{job="prometheus"}   # 内存占用
```

### 15.2 资源经验值

| 序列数 | 内存 | CPU | 磁盘 IOPS |
|-------:|-----:|----:|----------:|
| 10 万 | 2 GB | 1 核 | 低 |
| 50 万 | 6 GB | 2 核 | 中 |
| 100 万 | 12 GB | 4 核 | 中高 |
| 500 万 | 40 GB+ | 8 核+ | 高，建议 NVMe |

粗略公式：**内存 ≈ 活跃序列数 × 10KB**。

### 15.3 慢就这么调

1. 高频复杂查询 → 转成 **recording rule**
2. 序列太多 → 检查基数，`metric_relabel_configs` 丢弃无用指标
3. 抓取超时 → 调大 `scrape_timeout`，或拆分 job
4. 磁盘 IO 高 → 换 SSD/NVMe，或调小保留期
5. 仪表盘卡 → 减少面板数、缩短默认时间范围、加 `$__rate_interval`

---

## 16. 故障排查

| 现象 | 排查 | 常见原因 |
|------|------|----------|
| Target 显示 DOWN | `curl http://target:9100/metrics` | 防火墙/exporter 没起/端口错 |
| 图上没数据 | Prometheus → Status → Targets | 抓取失败或 label 不匹配 |
| 告警不触发 | Status → Rules 看 `for` 状态 | 表达式返回空 / for 时间没到 |
| 告警触发但没收到 | Alertmanager Status | 路由不匹配 / webhook 挂了 / 被静默 |
| Grafana 无数据源 | 容器网络 | URL 要用服务名 `http://prometheus:9090` |
| 内存持续增长 | `prometheus_tsdb_head_series` | 基数爆炸 |
| 重启后数据丢失 | 检查 volume 挂载 | 没持久化 |
| WAL 目录暴涨 | 磁盘 IO 打满导致压缩失败 | 换更快的磁盘 |

```bash
# 完整排查命令集
docker compose logs -f prometheus --tail 100
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose exec prometheus promtool check rules /etc/prometheus/rules/*.yml
curl -s http://127.0.0.1:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health!="up") | {job:.labels.job, url:.scrapeUrl, err:.lastError}'
curl -s http://127.0.0.1:9090/api/v1/rules | jq '.data.groups[].rules[] | select(.state=="firing") | .name'
curl -s http://127.0.0.1:9093/api/v2/alerts | jq '.[] | {name:.labels.alertname, state:.status.state}'
amtool config routes test --config.file=alertmanager/alertmanager.yml severity=critical
```

---

## 17. FAQ

**Q1：Prometheus 和 Zabbix 怎么选？**
Prometheus 适合云原生、容器化、指标维度多变的场景，PromQL 表达力强。Zabbix 适合传统 IT 设备（交换机、SNMP、Windows 主机）、需要开箱即用 Agent 和内置模板的场景。两者可以共存。

**Q2：最少需要什么配置的机器？**
监控 5-10 台机器：2 核 4G + 50G SSD 足够。监控 50 台：建议 4 核 8G + 200G NVMe。磁盘速度比 CPU 更关键。

**Q3：scrape_interval 设多少合适？**
默认 15s 适合大多数场景。改成 5s 存储和内存开销翻 3 倍，收益有限；改成 60s 会漏掉短时尖峰。**不要为了省存储把间隔调到分钟级**，会导致 `rate()` 失真。

**Q4：为什么我的告警一直 PENDING 不 FIRING？**
`for` 字段要求表达式**连续满足**指定时长。中间只要有一次评估不满足就重新计时。检查指标是否有抖动，或把 `for` 时间调短。

**Q5：怎么监控 Windows 服务器？**
用 `windows_exporter`（原 wmi_exporter），装成 Windows 服务，暴露 9182 端口，然后在 Prometheus 里正常加 target 即可。

**Q6：告警太多怎么治理？**
四步：① 统计过去 30 天每条告警的触发次数和"真实故障率"；② 触发多但从没导致故障的，提高阈值或删掉；③ 给同类告警配 `inhibit_rules`；④ 只有 critical 走即时通知，warning 进日报。**目标是让每条收到的告警都值得看一眼。**

**Q7：能不能不用 Docker？**
可以，各组件都是单二进制文件，下载后配 systemd unit 即可。但 Docker Compose 版本管理和升级更省心，本仓库以 compose 为主。

**Q8：监控数据要不要备份？**
监控数据通常不是核心资产，丢了重新采集即可。**但 Grafana 的仪表盘和 Prometheus 的规则文件一定要进 Git**，那才是真正积累下来的东西。

**Q9：多台 VPS 分布在不同机房怎么监控？**
两种方案：① 中心 Prometheus 直接抓所有节点（需要打通网络，用 WireGuard 组内网最省事）；② 每个机房一个 Prometheus，中心用 Thanos/Mimir 聚合查询。10 台以内选方案一。

**Q10：监控发现 CPU steal 很高怎么办？**
这是宿主机超售，你的代码优化没用。留存证据（Grafana 截图）找服务商，或者直接换机器。选机前可以参考 [vpsvip.net](https://vpsvip.net) 上的实测数据。

---

## 18. 相关资源

### 部署命令

```bash
git clone https://github.com/clashforwindows-net/server-monitoring.git
cd server-monitoring
cp .env.example .env && vim .env
docker compose up -d

# 访问（建议通过 SSH 隧道，避免暴露公网）
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 ops@your-server
# 浏览器打开 http://127.0.0.1:3000  (admin / 你在 .env 里设的密码)
```

### 添加一台新机器

```bash
# 1. 目标机器上启动 node_exporter
docker run -d --name node-exporter --restart unless-stopped \
  --net host --pid host -v /:/host:ro,rslave \
  prom/node-exporter:v1.8.1 --path.rootfs=/host

# 2. 监控机上追加 target（无需重启 Prometheus）
cat >> prometheus/targets/nodes.yml <<'EOF'
- targets: ['10.0.1.15:9100']
  labels: {env: prod, role: web}
EOF

# 3. 30 秒后自动生效，Status → Targets 确认
```

### 本组织相关仓库

| 仓库 | 说明 |
|------|------|
| [vps-scripts](https://github.com/clashforwindows-net/vps-scripts) | 自动化运维脚本库（告警脚本可对接本方案） |
| [vps-tools](https://github.com/clashforwindows-net/vps-tools) | 命令行诊断工具集与排查方法论 |
| [linux-server-20260402](https://github.com/clashforwindows-net/linux-server-20260402) | Linux 运维实战手册 |
| [vps-security-20260327](https://github.com/clashforwindows-net/vps-security-20260327) | 安全加固完全手册 |
| [vps-bench-20260327](https://github.com/clashforwindows-net/vps-bench-20260327) | VPS 性能测试与评分体系 |

### 官方文档

- Prometheus: https://prometheus.io/docs/
- PromQL: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Alertmanager: https://prometheus.io/docs/alerting/latest/configuration/
- Grafana: https://grafana.com/docs/grafana/latest/
- Awesome Prometheus Alerts: https://samber.github.io/awesome-prometheus-alerts/

### 关键词

`Prometheus监控` `Grafana仪表盘` `Alertmanager告警` `node_exporter` `cAdvisor` `blackbox_exporter` `PromQL` `SLO错误预算` `Loki日志` `Docker Compose监控` `服务器监控` `容量规划` `基数控制` `告警治理`

---

## Sponsor

监控能告诉你机器什么时候出问题，但选一台不会天天出问题的机器更重要。关注 CPU steal、磁盘 P99 延迟、网络稳定性：

- 🖥️ **VPS 评测与推荐**：[https://vpsvip.net](https://vpsvip.net)
- 🚀 **网络加速方案**：[ClashVIP](https://clashvip.net)
- 🧭 **工具资源导航**：[nav.clashvip.net](https://nav.clashvip.net)

**License**: MIT · PR welcome — 欢迎提交新的告警规则与仪表盘。

---

*Last updated: 2026-08-10 | Maintainer: clashforwindows-net | Sponsor: [VPS推荐](https://vpsvip.net)*
