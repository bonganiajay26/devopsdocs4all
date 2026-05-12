# Phase 7: Monitoring & Observability

---

## 7.1 The Three Pillars of Observability

```
OBSERVABILITY = METRICS + LOGS + TRACES
                    │          │        │
                    ▼          ▼        ▼
             "Is it slow?"  "Why?"  "Where?"
             Prometheus     ELK     Jaeger
             Grafana        Loki    Tempo
```

**Monitoring vs Observability:**

| Monitoring | Observability |
|-----------|--------------|
| Known unknowns | Unknown unknowns |
| "Is CPU > 80%?" | "Why is p99 latency high for user X?" |
| Pre-defined dashboards | Ad-hoc exploration |
| Alerts on thresholds | Understand system state from outputs |
| "Is the system up?" | "What is the system doing?" |

**The Four Golden Signals (Google SRE Book):**
1. **Latency** — time to serve a request (separate success vs error latency)
2. **Traffic** — demand on the system (requests/sec, queries/sec)
3. **Errors** — rate of requests that fail (explicit: 500s; implicit: wrong data)
4. **Saturation** — how "full" the service is (CPU, memory, queue depth)

---

## 7.2 Prometheus — Metrics at Scale

### What is Prometheus?

Prometheus is an open-source systems monitoring and alerting toolkit. It collects **time-series metrics** by **scraping** HTTP endpoints, stores them in its custom TSDB, and provides a powerful query language (PromQL) for analysis.

### Prometheus Internal Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PROMETHEUS SERVER                            │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    RETRIEVAL (Scraper)                       │    │
│  │  Job: kubernetes-nodes                                       │    │
│  │  Targets: [10.0.0.1:9100, 10.0.0.2:9100, ...]              │    │
│  │  Scrape interval: 15s                                        │    │
│  │  HTTP GET /metrics → text exposition format                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                             │
│                         ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              SERVICE DISCOVERY                               │    │
│  │  Kubernetes SD: watches API → discovers pods/nodes          │    │
│  │  File SD: reads targets from JSON/YAML files                │    │
│  │  EC2 SD, Consul SD, DNS SD, static_configs                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                             │
│                         ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              TSDB (Time Series Database)                     │    │
│  │                                                               │    │
│  │  head block (in memory, last 2h)                             │    │
│  │  ├── WAL (Write-Ahead Log) for durability                    │    │
│  │  └── chunks in memory                                        │    │
│  │                                                               │    │
│  │  persistent blocks (on disk, 2h - retention period)         │    │
│  │  ├── chunks/ (compressed time series data)                   │    │
│  │  ├── index (series → chunk mapping)                          │    │
│  │  └── meta.json                                               │    │
│  │                                                               │    │
│  │  Compaction: small blocks → larger blocks (efficiency)       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              RULES ENGINE                                    │    │
│  │  Recording Rules: pre-compute expensive queries              │    │
│  │  Alerting Rules: evaluate → fire to Alertmanager            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────┐                      ┌──────────────────────────┐  │
│  │ HTTP API     │                      │  ALERTMANAGER            │  │
│  │ /api/v1/     │                      │  Dedup → Group → Route   │  │
│  │ query        │                      │  → PagerDuty/Slack/Email  │  │
│  │ query_range  │                      └──────────────────────────┘  │
│  │ series       │                                                      │
│  └─────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────┘
          ▲                ▲              ▲
   scrape /metrics    scrape /metrics   Pushgateway
          │                │              │
   ┌──────────┐     ┌──────────┐    ┌──────────┐
   │ Node      │     │  App     │    │ Batch    │
   │ Exporter  │     │ /metrics │    │ Jobs     │
   │ (port     │     │ (port    │    │(push,    │
   │  9100)    │     │  8080)   │    │ not pull)│
   └──────────┘     └──────────┘    └──────────┘
```

### Metric Types

```
COUNTER — monotonically increasing (never decreases, resets to 0 on restart)
  http_requests_total{method="GET", status="200"} 1234567
  Use for: request counts, error counts, bytes transferred
  Always use _total suffix

GAUGE — can go up and down
  memory_usage_bytes{container="api"} 536870912
  Use for: current memory, active connections, queue size, temperature

HISTOGRAM — samples observations, counts them in configurable buckets
  http_request_duration_seconds_bucket{le="0.1"} 24054  ← ≤100ms
  http_request_duration_seconds_bucket{le="0.5"} 33444  ← ≤500ms
  http_request_duration_seconds_bucket{le="1.0"} 33478
  http_request_duration_seconds_bucket{le="+Inf"} 33480  ← all requests
  http_request_duration_seconds_sum 144758.0          ← total seconds
  http_request_duration_seconds_count 33480           ← total requests
  Use for: latency, request sizes (allows percentile calculation)

SUMMARY — similar to histogram but calculates quantiles client-side
  rpc_duration_seconds{quantile="0.5"}  0.012
  rpc_duration_seconds{quantile="0.99"} 0.085
  Use for: latency when you need precise quantiles
  ⚠️ Cannot aggregate across instances (use histogram instead in k8s)
```

### Prometheus Text Exposition Format

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",handler="/api/users",status="200"} 1234
http_requests_total{method="GET",handler="/api/users",status="404"} 56
http_requests_total{method="POST",handler="/api/users",status="201"} 789

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{handler="/api/users",le="0.005"} 100
http_request_duration_seconds_bucket{handler="/api/users",le="0.01"}  200
http_request_duration_seconds_bucket{handler="/api/users",le="0.025"} 400
http_request_duration_seconds_bucket{handler="/api/users",le="+Inf"} 1000
http_request_duration_seconds_sum{handler="/api/users"} 125.0
http_request_duration_seconds_count{handler="/api/users"} 1000
```

### PromQL — Query Language Deep Dive

```promql
# ===== INSTANT VECTORS =====
# Current value of a metric
http_requests_total                                # All time series
http_requests_total{status="200"}                  # Filter by label
http_requests_total{status=~"2.."}                 # Regex: 200, 201, etc
http_requests_total{status!="500"}                 # Not equal

# ===== RANGE VECTORS =====
# Values over a time range [5m = last 5 minutes]
http_requests_total[5m]                            # Used with rate/irate

# ===== FUNCTIONS =====

# rate() — per-second rate (smoothed over range, handles counter resets)
rate(http_requests_total[5m])                      # Req/sec over 5min window

# irate() — instantaneous rate (last 2 samples, more responsive)
irate(http_requests_total[5m])                     # Use for spiky metrics

# increase() — total increase over range
increase(http_requests_total[1h])                  # Total requests in 1h

# sum() — aggregate across all instances
sum(rate(http_requests_total[5m]))
sum by (status) (rate(http_requests_total[5m]))    # Group by status code
sum without (pod) (rate(http_requests_total[5m]))  # Remove pod label

# histogram_quantile() — calculate percentiles from histograms
histogram_quantile(0.99,                           # p99 latency
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

histogram_quantile(0.50,                           # Median latency
  sum by (le, handler) (                           # Per handler
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# ===== PRACTICAL EXAMPLES =====

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# CPU usage by pod
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="production"}[5m])
) * 100

# Memory usage (bytes)
sum by (pod) (
  container_memory_working_set_bytes{namespace="production"}
)

# Request rate per endpoint
sum by (handler) (
  rate(http_requests_total[5m])
)

# Apdex score (Application Performance Index)
# Satisfied: < 0.3s, Tolerating: < 1.2s, Frustrated: >= 1.2s
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m]))
) / 2
/
sum(rate(http_request_duration_seconds_count[5m]))
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # Default: scrape every 15s
  evaluation_interval: 15s      # Evaluate rules every 15s
  scrape_timeout: 10s
  
  external_labels:              # Labels added to all time series
    cluster: production
    region: us-east-1

rule_files:
  - "rules/*.yml"               # Alert and recording rules

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # Kubernetes Pods auto-discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (\d+)
        replacement: $1
      # Keep only useful labels
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
  
  # Node exporters
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    
  # Static targets
  - job_name: 'postgres-exporter'
    static_configs:
      - targets: ['postgres-exporter:9187']
    metrics_path: /metrics
    scheme: https
    tls_config:
      ca_file: /etc/ssl/certs/ca-cert.pem
```

### Alerting Rules

```yaml
# rules/alerts.yml
groups:
  - name: application
    interval: 30s      # Override global evaluation_interval
    rules:
      # Recording rule: pre-compute expensive query
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      # Alerting rule
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          * 100 > 5
        for: 5m          # Must be true for 5min before firing
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High HTTP error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | printf \"%.2f\" }}% (threshold: 5%)"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le, job) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency {{ $value | printf \"%.3f\" }}s on {{ $labels.job }}"
      
      - alert: PodNotReady
        expr: kube_pod_status_ready{condition="true"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
      
      - alert: DiskSpaceRunningLow
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"} 
           / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}: {{ $value | printf \"%.1f\" }}% free"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'job']
  group_wait: 30s          # Wait to batch alerts
  group_interval: 5m       # How long before resending
  repeat_interval: 4h      # Resend if not resolved
  receiver: 'slack-general'
  
  routes:
    - match:
        severity: critical
      receiver: pagerduty
      continue: true   # Also send to next matching route
    
    - match:
        severity: critical
        team: backend
      receiver: slack-backend-critical
      group_wait: 0s   # Page immediately for critical
    
    - match_re:
        alertname: ^(Watchdog|InfoInhibitor)$
      receiver: 'null'  # Silence informational alerts

receivers:
  - name: 'null'
  
  - name: slack-general
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
  
  - name: pagerduty
    pagerduty_configs:
      - service_key: '{{ .ExternalURL }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'

inhibit_rules:
  # If cluster is down, don't alert for individual components
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '.+'
    equal: ['cluster']
```

---

## 7.3 Grafana — Visualization

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                    GRAFANA                           │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Web UI      │  │  Backend API  │  │  Database  │ │
│  │  (React)     │  │  (Go)         │  │  (SQLite/  │ │
│  │              │  │               │  │  MySQL/PG) │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
│                         │                             │
│              ┌──────────┼──────────┐                 │
│              ▼          ▼          ▼                 │
│        Dashboards   Alerting   Users/Orgs            │
│        Panels       Rules      Permissions           │
│        Variables    Contacts   API Keys              │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │              DATA SOURCE PLUGINS             │    │
│  │  Prometheus │ Loki │ Tempo │ ES │ InfluxDB  │    │
│  │  CloudWatch │ MySQL │ PostgreSQL │ ...       │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
         │           │           │
   Prometheus      Loki        Tempo
   (metrics)      (logs)      (traces)
```

### Dashboard as Code (Grafonnet/JSON)

```json
// dashboard.json — programmatic dashboard
{
  "title": "Application Overview",
  "uid": "app-overview",
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "multi": false,
        "includeAll": false
      }
    ]
  },
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum by (status) (rate(http_requests_total{namespace=\"$namespace\"}[5m]))",
          "legendFormat": "{{status}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "custom": {"lineWidth": 2}
        }
      }
    }
  ]
}
```

---

## 7.4 ELK Stack — Log Management

### Architecture Overview

```
APPLICATION LOGS
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│                     LOG SHIPPERS (Beat agents)                   │
│                                                                   │
│  Filebeat (log files)    Metricbeat (system metrics)            │
│  Packetbeat (network)    Heartbeat (uptime)                      │
│                                                                   │
│  Lightweight Go agents — installed on every server              │
│  Read from file/syslog/stdin → ship to Logstash or ES directly  │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 5044 / Beats protocol or port 5000 / syslog)
┌─────────────────────────────────────────────────────────────────┐
│                         LOGSTASH                                 │
│                                                                   │
│  INPUT → FILTER → OUTPUT pipeline                               │
│                                                                   │
│  Input:   beats, syslog, kafka, http, stdin                     │
│  Filter:  grok (parse unstructured), mutate (rename/remove),    │
│           geoip (add location), date (parse timestamps)         │
│  Output:  elasticsearch, kafka, s3, stdout                      │
│                                                                   │
│  Horizontally scalable (multiple Logstash nodes)                │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 9200 / HTTP or port 9300 / TCP cluster comms)
┌─────────────────────────────────────────────────────────────────┐
│                      ELASTICSEARCH                               │
│                                                                   │
│  Node 1 (Master)  ←→  Node 2 (Data)  ←→  Node 3 (Data)        │
│                                                                   │
│  Index: logs-2024.01.15  (daily rolling index)                  │
│  ├── Primary shard 0 (on Node 2)                                │
│  ├── Primary shard 1 (on Node 3)                                │
│  ├── Replica shard 0 (on Node 3)  ← replica of primary 0       │
│  └── Replica shard 1 (on Node 2)  ← replica of primary 1       │
│                                                                   │
│  Inverted index: word → [document IDs] (fast full-text search)  │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼ (port 5601 / HTTP)
┌─────────────────────────────────────────────────────────────────┐
│                         KIBANA                                   │
│  Discover: browse/search logs                                    │
│  Dashboard: visualizations                                       │
│  APM: application performance                                    │
│  Alerting: log-based alerts                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Logstash Pipeline

```ruby
# /etc/logstash/conf.d/application.conf

input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
  }
}

filter {
  # Parse JSON logs (for structured logging apps)
  if [fields][log_type] == "application" {
    json {
      source => "message"
      target => "app"
    }
    
    # Parse timestamp
    date {
      match => ["[app][timestamp]", "ISO8601"]
      target => "@timestamp"
    }
    
    # Rename fields
    mutate {
      rename => { "[app][level]" => "log_level" }
      rename => { "[app][trace_id]" => "trace_id" }
      add_field => { "environment" => "%{[fields][environment]}" }
    }
  }
  
  # Parse nginx access logs (unstructured)
  if [fields][log_type] == "nginx" {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{USER:auth} \[%{HTTPDATE:timestamp}\] "%{WORD:http_method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code:int} %{NUMBER:body_bytes:int} "%{GREEDYDATA:referrer}" "%{GREEDYDATA:user_agent}" %{NUMBER:request_time:float}'
      }
    }
    
    geoip {
      source => "client_ip"
      target => "geoip"
    }
    
    useragent {
      source => "user_agent"
      target => "ua"
    }
  }
  
  # Drop health check noise
  if [request] =~ "/health" {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "logstash_writer"
    password => "${LOGSTASH_PASSWORD}"
    ssl => true
    cacert => "/etc/logstash/certs/ca.crt"
    
    # Daily rolling indices
    index => "logs-%{[fields][log_type]}-%{+YYYY.MM.dd}"
    
    # Use ILM for lifecycle management
    ilm_enabled => true
    ilm_rollover_alias => "logs-app"
    ilm_policy => "logs-30d-policy"
  }
}
```

### Elasticsearch Index Lifecycle Management (ILM)

```json
// PUT _ilm/policy/logs-30d-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",        // Rollover after 1 day
            "max_size": "50gb"      // or when 50GB
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "3d",           // Move to warm after 3 days
        "actions": {
          "shrink": { "number_of_shards": 1 },  // Reduce shards
          "forcemerge": { "max_num_segments": 1 }, // Optimize
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "7d",           // Move to cold after 7 days
        "actions": {
          "freeze": {},            // Read-only, minimal resources
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "30d",          // Delete after 30 days
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 7.5 Loki — Logs for Kubernetes

### Why Loki Instead of ELK?

```
ELK:                                    LOKI:
Full-text indexes EVERYTHING            Indexes ONLY labels (like Prometheus)
$$$$ storage (indexes ~10x log size)    $ storage (compressed raw logs)
Complex setup                           Simple (runs in K8s easily)
Powerful search                         LogQL (same syntax as PromQL)
Best for: structured log analysis       Best for: K8s operational logs
```

### Loki Architecture

```
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Promtail   │  │  Promtail  │  │  Promtail  │
│  (DaemonSet)│  │  (DaemonSet│  │  (DaemonSet│
│  K8s Node 1 │  │  Node 2    │  │  Node 3    │
│  Tails pod  │  │  Tails pod │  │  Tails pod │
│  log files  │  │  log files │  │  log files │
└────────────┘  └────────────┘  └────────────┘
       │                │               │
       └────────────────┼───────────────┘
                        │
                        ▼ (port 3100 / HTTP)
                 ┌─────────────┐
                 │    LOKI     │
                 │             │
                 │  Distributor│ ← Receives writes, hashes stream
                 │      │      │
                 │  Ingester   │ ← In-memory recent logs + WAL
                 │      │      │
                 │  Querier    │ ← Executes LogQL queries
                 │      │      │
                 │  Compactor  │ ← Manages long-term storage
                 └─────────────┘
                        │
                 ┌──────┴──────┐
                 │  S3/GCS/    │  ← Chunks (compressed log data)
                 │  Azure Blob │  ← Index (label → chunk mapping)
                 └─────────────┘
```

### LogQL — Query Language

```logql
# ===== LOG QUERIES =====

# Simple label filter
{namespace="production", app="api-server"}

# Multiple labels
{namespace="production"} |= "ERROR"    # Contains "ERROR"
{namespace="production"} != "DEBUG"    # Exclude "DEBUG"
{namespace="production"} |~ "error|Error|ERROR"  # Regex

# Parse JSON logs
{app="api"} | json | status_code >= 500

# Parse with pattern
{app="nginx"} 
  | pattern `<ip> - - [<ts>] "<method> <path> HTTP/<_>" <status> <size>`
  | status >= 500

# Pipeline filters
{namespace="production"}
  | json
  | line_format "{{.level}} {{.message}}"    # Reformat output
  | label_format severity=level              # Rename labels

# ===== METRIC QUERIES =====

# Log rate
rate({app="api"} |= "error" [5m])    # Errors per second

# Count by status code
sum by (status) (
  rate({app="nginx"} | json | __error__="" [5m])
)

# Error rate percentage  
sum(rate({app="api"} |= "ERROR" [5m]))
/
sum(rate({app="api"} [5m]))
* 100

# p99 latency from logs
quantile_over_time(0.99,
  {app="api"} | json | unwrap response_time [5m]
) by (endpoint)
```

---

## 7.6 Distributed Tracing — Jaeger & OpenTelemetry

### Why Distributed Tracing?

```
User request → API Gateway → Auth Service → User Service → DB
                                         → Order Service → Inventory Service → DB
                                                         → Payment Service → External API

PROBLEM: Request is slow. Which service is slow?
SOLUTION: Trace the request through all services with timing data.

TRACE: A complete request from start to finish
  └── SPAN: A unit of work (one service, one DB query)
        ├── trace_id: abc123 (same across all services)
        ├── span_id: xyz789 (unique per span)
        ├── parent_span_id: parent (links spans in tree)
        ├── operation: "HTTP GET /api/orders"
        ├── service: "order-service"
        ├── start_time: 1704067200000
        ├── duration: 45ms
        └── tags: {http.status=200, db.type=postgresql}
```

### OpenTelemetry — The Standard

```
OpenTelemetry (OTel) is the CNCF standard for:
  - Metrics (replaces Prometheus client libraries)
  - Logs (structured logging)  
  - Traces (replaces Jaeger/Zipkin client libraries)

Architecture:
  App → OTel SDK → OTel Collector → Jaeger/Tempo/DataDog/etc.
  
Benefits:
  - Vendor-neutral: switch backends without changing app code
  - Auto-instrumentation: zero-code instrumentation for common frameworks
  - W3C TraceContext: standard trace propagation headers
```

```python
# Python app with OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Setup
provider = TracerProvider(
    resource=Resource.create({
        "service.name": "order-service",
        "service.version": "1.2.0",
        "deployment.environment": "production"
    })
)
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://otel-collector:4317")
    )
)
trace.set_tracer_provider(provider)

# Auto-instrument FastAPI and SQLAlchemy
FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument(engine=engine)

# Manual instrumentation
tracer = trace.get_tracer(__name__)

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        
        # This creates a child span automatically
        order = await db.query(Order).filter_by(id=order_id).first()
        
        if not order:
            span.set_attribute("error", True)
            span.set_status(trace.StatusCode.ERROR, "Order not found")
            raise HTTPException(404)
        
        # Propagate trace to downstream service
        inventory = await inventory_client.get_inventory(order.items)
        
        span.add_event("order_retrieved", {"item_count": len(order.items)})
        return order
```

### OpenTelemetry Collector

```yaml
# otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"
  
  # Collect Prometheus metrics too
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          static_configs:
            - targets: ['localhost:8888']

processors:
  batch:                          # Batch for efficiency
    send_batch_size: 10000
    timeout: 10s
  
  memory_limiter:                 # Prevent OOM
    check_interval: 1s
    limit_mib: 512
  
  resource:                       # Add resource attributes
    attributes:
      - key: environment
        value: production
        action: upsert

exporters:
  jaeger:
    endpoint: "jaeger:14250"
    tls:
      insecure: true
  
  prometheus:
    endpoint: "0.0.0.0:8889"     # Prometheus scrapes this
  
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
  
  logging:                        # Debug: print to stdout
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

---

## 7.7 Production Observability Stack Setup

### Complete Kubernetes Monitoring Stack

```yaml
# Using kube-prometheus-stack Helm chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring-values.yaml
```

```yaml
# monitoring-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
    
    # Scrape all ServiceMonitors in all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    
    # Remote write to Thanos/Cortex for long-term storage
    remoteWrite:
      - url: https://thanos-receive:10908/api/v1/receive

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 10Gi
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/...'
    route:
      receiver: slack

grafana:
  adminPassword: "changeme"
  persistence:
    enabled: true
    size: 10Gi
  
  # Auto-provision dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
  
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://kube-prometheus-stack-prometheus:9090
          isDefault: true
        - name: Loki
          type: loki
          url: http://loki:3100
        - name: Tempo
          type: tempo
          url: http://tempo:3100
```

### Service Monitor — Auto-Discover App Metrics

```yaml
# servicemonitor.yaml — tells Prometheus to scrape your app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app          # Selects Services with this label
  endpoints:
    - port: metrics        # Port name in the Service
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
      - production
```

---

## 7.8 Observability Interview Questions

### Beginner

**Q: What is the difference between monitoring and observability?**

> Monitoring tells you WHEN something is wrong by alerting on pre-defined thresholds. Observability tells you WHY something is wrong by providing the ability to ask arbitrary questions about system state using metrics, logs, and traces. A highly observable system lets you debug novel failure modes you've never seen before without deploying new instrumentation.

**Q: What are the four golden signals?**

> Latency, Traffic, Errors, and Saturation. These four signals, from Google's SRE Book, give you enough information to detect virtually any user-visible problem. Latency: how long requests take (separately track error latency). Traffic: demand on the system (req/sec). Errors: rate of failed requests. Saturation: how "full" the service is (CPU %, queue depth, disk usage).

### Advanced

**Q: Explain how Prometheus handles counter resets.**

> When a process restarts, counters reset to 0. The `rate()` function detects resets: if a sample is smaller than the previous sample, it assumes a reset occurred and adjusts the calculation accordingly. This is why you should always use `rate()` or `increase()` on counters, never raw counter values in graphs.

**Q: How would you debug high p99 latency that only affects 1% of users?**

> 1. **Identify the pattern**: Use histogram_quantile with breakdowns by endpoint, region, user_id to isolate the scope. 2. **Correlate with traces**: Find slow traces in Jaeger/Tempo that match the time window — they'll show which span is slow. 3. **Check logs**: Correlate trace_id from slow traces with logs to find errors or slow queries. 4. **Hypotheses**: Hot partition in DB (specific user's data on slow shard), GC pauses, connection pool exhaustion for specific backends, CDN miss patterns, specific payload sizes triggering slow code paths. 5. **Resolution**: Depends on root cause — add DB indexes, tune JVM GC, increase connection pool, add caching layer.

**Q: Design an observability stack for a 100-microservice system.**

> - **Metrics**: Prometheus per cluster + Thanos for global view and long-term storage (2 years). Alertmanager with PagerDuty integration, tiered severity routing.
> - **Logs**: Loki (or ELK for compliance/search requirements). Structured JSON logging standard enforced. Log correlation by trace_id.
> - **Traces**: OTel collector as sidecar/DaemonSet, Tempo or Jaeger for storage. 1% sampling for normal traffic, 100% for errors.
> - **Dashboards**: Grafana with USE method (Utilization, Saturation, Errors) per service, RED method (Rate, Errors, Duration) for APIs, auto-provisioned via GitOps.
> - **SLOs**: Pyrra or Sloth for SLO tracking. Error budget burn rate alerts.
> - **On-call**: Runbooks linked from every alert. Incident.io or PagerDuty for rotation.
```
