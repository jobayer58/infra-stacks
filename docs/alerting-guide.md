# Prometheus Alerting Configuration Guide

> Transform your monitoring from **reactive** to **proactive** with intelligent alerts.

This guide demonstrates how to configure Prometheus Alert Rules to proactively monitor your Docker infrastructure, detecting issues before they impact your users.

## Why Alerts Matter

Alerts enable **proactive problem detection** rather than reactive dashboard monitoring.

**Reactive (without alerts):**
```
Server CPU at 90% → You discover it on dashboard → Application becomes slow → Users complain
```

**Proactive (with alerts):**
```
Server CPU reaches 80% → Prometheus triggers alert → You receive notification → Investigate BEFORE user impact
```

## Alert Architecture

```
Prometheus (evaluates rules)
    ↓
AlertManager (centralizes alerts)
    ↓
Notification Channels (Email, Slack, Discord, PagerDuty)
```

## Alert Rule Structure

An alert rule has four key components:

```yaml
groups:
  - name: my_group
    interval: 1m  # Evaluate condition every 1 minute
    rules:
      - alert: HighCPUUsage
        expr: cpu_usage > 80  # PromQL query
        for: 5m  # Fire alert after condition is true for 5 minutes
        labels:
          severity: warning  # or critical
        annotations:
          summary: "High CPU detected"
          description: "CPU usage at {{ $value }}%"
```

### Component Details

| Component | Example | Purpose |
|-----------|---------|----------|
| **expr** | `node_cpu > 80` | PromQL query; if > 0, alert activates |
| **for** | `5m` | Wait 5 minutes of condition being true before firing (prevents false positives) |
| **severity** | `warning`, `critical` | Alert level; routes to different channels |
| **summary** | `"High CPU usage"` | Brief alert title |
| **description** | `"CPU at {{ $value }}%"` | Details; `{{ }}` are dynamic variables |

## Production-Ready Alert Rules

### 1. High CPU Usage (WARNING: 75%, CRITICAL: 90%)

**When to use:** Always. High CPU indicates bottleneck.

**Impact:** Slow application, timeouts, failing requests.

```yaml
- alert: HighCPUUsage
  expr: |
    (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 75
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High CPU on {{ $labels.instance }}"
    description: "CPU usage: {{ printf \"%.2f\" $value }}% (threshold: 75%)"

- alert: CriticalCPUUsage
  expr: |
    (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 90
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "CRITICAL CPU on {{ $labels.instance }}"
    description: "CPU usage: {{ printf \"%.2f\" $value }}% (threshold: 90%) - IMMEDIATE ACTION REQUIRED"
```

### 2. High Memory Usage (WARNING: 80%, CRITICAL: 95%)

**When to use:** Always. High memory indicates memory leak or insufficient resources.

**Impact:** OOM Killer terminates processes, container restart, downtime.

```yaml
- alert: HighMemoryUsage
  expr: |
    (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High memory on {{ $labels.instance }}"
    description: "Memory usage: {{ printf \"%.2f\" $value }}% (threshold: 80%)"

- alert: CriticalMemoryUsage
  expr: |
    (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 95
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "CRITICAL MEMORY on {{ $labels.instance }}"
    description: "Memory usage: {{ printf \"%.2f\" $value }}% (threshold: 95%) - OOM RISK"
```

### 3. Low Disk Space (WARNING: 15%, CRITICAL: 5%)

**When to use:** Always. Full disk breaks logs, databases, and causes crashes.

**Impact:** Application cannot write data, database corruption, container failure.

```yaml
- alert: LowDiskSpace
  expr: |
    (node_filesystem_avail_bytes{fstype!="tmpfs",fstype!="fuse.lxcfs"} / node_filesystem_size_bytes) * 100 < 15
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Low disk space on {{ $labels.instance }} ({{ $labels.mountpoint }})"
    description: "Free space: {{ printf \"%.2f\" $value }}% (threshold: 15%)"

- alert: CriticalDiskSpace
  expr: |
    (node_filesystem_avail_bytes{fstype!="tmpfs",fstype!="fuse.lxcfs"} / node_filesystem_size_bytes) * 100 < 5
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "CRITICAL DISK SPACE on {{ $labels.instance }} ({{ $labels.mountpoint }})"
    description: "Free space: {{ printf \"%.2f\" $value }}% (threshold: 5%) - IMMEDIATE ACTION REQUIRED"
```

### 4. Container Restarting Frequently

**When to use:** Always in production. Frequent restarts indicate crash or health issue.

**Impact:** Unstable service, intermittent downtime, potential data loss.

```yaml
- alert: ContainerRestarting
  expr: |
    increase(container_last_seen{name!=""}[5m]) > 2
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Container restarting frequently: {{ $labels.name }}"
    description: "Container {{ $labels.name }} restarted {{ $value | humanize }} times in 5 minutes. Check logs: docker logs {{ $labels.name }}"
```

**Requires:** cAdvisor in docker-compose.yml:
```yaml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
  ports:
    - "8080:8080"
  networks:
    - monitoring
```

### 5. Prometheus Scrape Failures

**When to use:** Always. If Prometheus cannot scrape metrics, alerting is blind.

**Impact:** Complete monitoring blindness, no visibility into system health.

```yaml
- alert: PrometheusScrapeFailing
  expr: |
    up == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus scrape failure: {{ $labels.instance }}"
    description: "Job {{ $labels.job }} not responding. Check: http://{{ $labels.instance }}:{{ $labels.port }}"
```

### 6. Prometheus Low Disk Space

**When to use:** If running Prometheus locally (without Thanos/remote storage).

**Impact:** Prometheus deletes old metrics, loses historical data.

```yaml
- alert: PrometheusLowDiskSpace
  expr: |
    predict_linear(node_filesystem_avail_bytes{mountpoint="/prometheus"}[1h], 24*3600) < 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus will run out of disk space in ~24 hours"
    description: "Free space at /prometheus: {{ humanizeBytes $value }}. Consider data cleanup."
```

### 7. Node Exporter Offline

**When to use:** When monitoring multiple servers. Detects when servers go offline.

**Impact:** Cannot monitor server, problems invisible.

```yaml
- alert: NodeExporterDown
  expr: |
    up{job="node-exporter"} == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Node Exporter offline: {{ $labels.instance }}"
    description: "Node Exporter at {{ $labels.instance }} not responding for 2 minutes. Server may be down or process crashed."
```

### 8. High Swap Memory Usage

**When to use:** On servers with swap enabled. High swap usage indicates insufficient real RAM.

**Impact:** Degraded performance (swap is very slow), may lead to OOM.

```yaml
- alert: HighSwapUsage
  expr: |
    (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100 < 20
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High swap usage on {{ $labels.instance }}"
    description: "Free swap: {{ printf \"%.2f\" $value }}%. System may have insufficient real RAM."
```

### 9. High Network I/O

**When to use:** If concerned about DDoS or abnormal traffic.

**Impact:** Network saturation, packet loss, application unreachable.

```yaml
- alert: HighNetworkTraffic
  expr: |
    rate(node_network_receive_bytes_total{device!="lo"}[5m]) > 100000000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High network traffic on {{ $labels.instance }} ({{ $labels.device }})"
    description: "Receive rate: {{ humanize $value }}B/s (threshold: 100MB/s)"
```

### 10. High Inode Usage

**When to use:** If you have many small files (common in applications creating logs/temp files).

**Impact:** Cannot create new files even with free disk space available.

```yaml
- alert: HighInodeUsage
  expr: |
    (node_filesystem_files_free / node_filesystem_files) * 100 < 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High inode usage on {{ $labels.instance }} ({{ $labels.mountpoint }})"
    description: "Free inodes: {{ printf \"%.2f\" $value }}%. Clean up old files."
```

## Setup Instructions

### Step 1: Create Alert Rules File

```bash
mkdir -p stacks/monitoring/prometheus
```

Create `stacks/monitoring/prometheus/alert-rules.yml` and add the rules you need.

### Step 2: Register Rules with Prometheus

Update `stacks/monitoring/prometheus/prometheus.yml`:

```yaml
rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

### Step 3: Add AlertManager to docker-compose.yml

```yaml
alertmanager:
  image: prom/alertmanager:latest
  container_name: alertmanager
  ports:
    - "9093:9093"
  volumes:
    - ./prometheus/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    - alertmanager_data:/alertmanager
  command:
    - '--config.file=/etc/alertmanager/alertmanager.yml'
    - '--storage.path=/alertmanager'
  networks:
    - monitoring

volumes:
  alertmanager_data:
```

### Step 4: Configure Notifications

Create `stacks/monitoring/prometheus/alertmanager.yml`:

#### Option A: Slack (Recommended)

1. Create webhook at https://api.slack.com/apps:
   - Create New App → From scratch
   - Name: "Prometheus Alerts"
   - Incoming Webhooks → On
   - Add New Webhook to Workspace
   - Copy webhook URL

2. Configure `alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'slack'
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
```

#### Option B: Discord

```yaml
receivers:
  - name: 'discord'
    webhook_configs:
      - url: 'https://discordapp.com/api/webhooks/YOUR/WEBHOOK/URL'
        send_resolved: true
```

#### Option C: Email (Gmail)

```yaml
global:
  smtp_from: 'prometheus@yourdomain.com'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'  # Generate at https://myaccount.google.com/apppasswords

receivers:
  - name: 'email'
    email_configs:
      - to: 'ops@yourdomain.com'
        from: 'prometheus@yourdomain.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'your-email@gmail.com'
        auth_password: 'your-app-password'
        headers:
          Subject: 'Alert: {{ .GroupLabels.alertname }}'
```

### Step 5: Restart Services

```bash
docker compose restart prometheus alertmanager
```

Verify at `http://localhost:9090/alerts` that rules are loaded.

## Best Practices

### 1. Start with Silent Notifications

Don't send to Slack/Email immediately. Test first:

```yaml
route:
  receiver: 'null'
receivers:
  - name: 'null'
```

After validating alerts trigger correctly, enable notifications.

### 2. Use "for" to Prevent False Positives

```yaml
for: 5m  # Only alert if condition sustained for 5 minutes
```

A 1-second CPU spike doesn't warrant an alert. Wait for sustained condition.

### 3. Always Include Context in Annotations

**Bad:**
```yaml
description: "High CPU"
```

**Good:**
```yaml
description: "CPU usage at {{ printf \"%.2f\" $value }}% on {{ $labels.instance }}. Debug with: docker top <container>"
```

### 4. Route Intelligently by Severity

```yaml
route:
  receiver: 'slack-general'
  routes:
    - match:
        severity: critical
      receiver: 'slack-oncall'  # On-call team
    - match:
        severity: warning
      receiver: 'slack-general'  # General channel
```

### 5. Tune Thresholds for Your Hardware

This guide uses generic values. Adjust for your infrastructure:
- 1vCPU VPS: CPU threshold 60% (1 core maxed = 100%)
- 4vCPU VPS: CPU threshold 75%
- 256GB RAM server: memory threshold 85% (still 40GB free)

### 6. Organize Rules in Groups

```yaml
groups:
  - name: system
    rules:
      - alert: HighCPU
        ...
  - name: docker
    rules:
      - alert: ContainerRestarting
        ...
  - name: prometheus
    rules:
      - alert: PrometheusDown
        ...
```

## Troubleshooting

### Alert Not Firing?

1. **Verify rule is loaded:**
   ```bash
   curl http://localhost:9090/api/v1/rules
   ```

2. **Test the PromQL expression in UI:**
   - Go to http://localhost:9090
   - Execute the PromQL query
   - Check if it returns values

3. **View active alerts:**
   ```bash
   curl http://localhost:9090/api/v1/alerts
   ```

4. **Check Prometheus logs:**
   ```bash
   docker compose logs prometheus
   ```

### AlertManager Not Sending Notifications?

1. **Test webhook manually:**
   ```bash
   curl -X POST 'WEBHOOK_URL' \
     -H 'Content-Type: application/json' \
     -d '{"text": "Test alert"}'
   ```

2. **Check AlertManager logs:**
   ```bash
   docker compose logs alertmanager
   ```

3. **View alerts in AlertManager UI:**
   ```bash
   http://localhost:9093
   ```

## Next Steps

- [ ] Implement escalation (notify manager if no response in 30 min)
- [ ] Add custom templates (logos, signatures)
- [ ] Integrate with PagerDuty (automatic on-call)
- [ ] Create runbooks (action guides for each alert)
- [ ] Configure automatic silencing during maintenance windows

## References

- [Prometheus Alerting Documentation](https://prometheus.io/docs/alerting/latest/overview/)
- [AlertManager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [PromQL Query Language](https://prometheus.io/docs/prometheus/latest/querying/basics/)
