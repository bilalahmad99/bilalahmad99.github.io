---
layout: default
---

## Observability with Grafana OSS Stack

A practical guide to building comprehensive observability with Grafana, Loki, Mimir, Tempo, and Prometheus, including  lessons learned and operational insights.

### The Grafana OSS Stack Overview

The Grafana OSS stack provides a complete observability solution with:
- **Prometheus**: Metrics collection and storage
- **Loki**: Log aggregation and storage
- **Tempo**: Distributed tracing
- **Mimir**: Prometheus-compatible metrics storage at scale
- **Grafana**: Visualization and alerting

### Basic Architecture Overview

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Prometheus │    │    Loki     │    │   Tempo     │
│   (Metrics) │    │   (Logs)    │    │  (Traces)   │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌─────────────┐
                    │   Grafana   │
                    │(Dashboards) │
                    └─────────────┘
```

### Prometheus Setup & Configuration

#### Basic Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

#### Kubernetes ServiceMonitor Example
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Loki Configuration & Log Collection

#### Loki Configuration
```yaml
# loki-config.yml
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
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

query_scheduler:
  max_outstanding_requests_per_tenant: 2048

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  max_cache_freshness_per_query: 10m
  split_queries_by_interval: 15m
  max_query_parallelism: 32
  max_streams_per_user: 0
  max_line_size: 256000
```

#### Promtail Configuration for Kubernetes
```yaml
# promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_controller_name]
        regex: ([0-9a-z-.]+?)(-[0-9a-f]{8,10})?
        action: replace
        target_label: __tmp_controller_name
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
        action: replace
        target_label: app_kubernetes_io_name
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
        action: replace
        target_label: app_kubernetes_io_instance
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
        action: replace
        target_label: app_kubernetes_io_component
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
    pipeline_stages:
      - cri: {}
```

### Tempo Configuration for Distributed Tracing

#### Tempo Configuration
```yaml
# tempo-config.yml
server:
  http_listen_port: 3200

distributor:
  receivers:
    jaeger:
      protocols:
        thrift_http:
        grpc:
        thrift_binary:
        thrift_compact:
    zipkin:
    otlp:
      protocols:
        grpc:
        http:
    opencensus:

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 1h

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/traces
    pool:
      max_workers: 100
      queue_depth: 10000
```

#### Application Instrumentation Example
```javascript
// Node.js application with OpenTelemetry
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### Mimir Configuration for Scale

#### Mimir Configuration
```yaml
# mimir-config.yml
target: all,alertmanager

server:
  http_listen_port: 9009
  grpc_listen_port: 9095

common:
  storage:
    backend: s3
    s3:
      endpoint: s3.amazonaws.com
      bucket_name: mimir-data
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}
      access_key_id: ${AWS_ACCESS_KEY_ID}
      s3forcepathstyle: false
      insecure: false

ingester:
  ring:
    kvstore:
      store: memberlist
  lifecycler:
    join_after: 0s
    num_tokens: 512
    ring:
      replication_factor: 3

distributor:
  ring:
    kvstore:
      store: memberlist

store_gateway:
  sharding_enabled: true

compactor:
  sharding_enabled: true

limits:
  max_global_series_per_user: 1000000
  max_global_series_per_metric: 100000
  ingestion_rate: 10000
  ingestion_burst_size: 20000
```

### Grafana Configuration & Dashboards

#### Grafana Configuration
```ini
# grafana.ini
[server]
http_port = 3000
domain = grafana.example.com
root_url = https://grafana.example.com

[security]
admin_user = admin
admin_password = ${GRAFANA_ADMIN_PASSWORD}

[auth.anonymous]
enabled = true
org_name = Main Org.

[datasources]
datasources.yaml:
  apiVersion: 1
  datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      access: proxy
      isDefault: true
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
    - name: Tempo
      type: tempo
      url: http://tempo:3200
      access: proxy
```

#### Sample Dashboard Configuration
```json
{
  "dashboard": {
    "title": "Application Observability",
    "panels": [
      {
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "refId": "A"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])",
            "refId": "A"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "stat",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "refId": "A"
          }
        ]
      }
    ]
  }
}
```

### Alerting Configuration

#### Prometheus Alert Rules
```yaml
# alert_rules.yml
groups:
  - name: application_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors per second"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }} seconds"
```

#### Grafana Alert Rules
```yaml
# grafana-alert-rules.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy

alert_rules:
  - uid: high-cpu-usage
    title: High CPU Usage
    condition: C
    data:
      - refId: A
        queryType: ''
        relativeTimeRange:
          from: 300
          to: 0
        datasource:
          type: prometheus
          uid: prometheus
        model:
          expr: '100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)'
          refId: A
    noDataState: NoData
    execErrState: Alerting
    for: 2m
    annotations:
      summary: 'High CPU usage detected'
```

### Lessons Learned from Production

#### 1. Storage Planning
**Lesson**: Plan storage capacity carefully - logs and metrics grow faster than expected.

```yaml
# Loki retention policy
limits_config:
  retention_period: 30d
  max_streams_per_user: 10000
  max_line_size: 256000

# Prometheus retention
global:
  external_labels:
    cluster: 'production'
  scrape_interval: 15s
  evaluation_interval: 15s

storage:
  tsdb:
    retention.time: 30d
    retention.size: 50GB
```

#### 2. Resource Limits
**Lesson**: Set proper resource limits to prevent OOM kills.

```yaml
# Kubernetes resource limits
resources:
  limits:
    memory: "2Gi"
    cpu: "1000m"
  requests:
    memory: "1Gi"
    cpu: "500m"
```

#### 3. Label Management
**Lesson**: Be careful with high-cardinality labels - they can explode storage costs.

```yaml
# Good: Low cardinality
labels:
  - job
  - instance
  - status

# Bad: High cardinality
labels:
  - user_id
  - request_id
  - timestamp
```

#### 4. Query Performance
**Lesson**: Use recording rules for expensive queries.

```yaml
# recording_rules.yml
groups:
  - name: recording_rules
    rules:
      - record: job:http_requests_per_second
        expr: rate(http_requests_total[5m])
      
      - record: job:http_request_duration_seconds:rate5m
        expr: rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

#### 5. Alert Fatigue Prevention
**Lesson**: Use alert grouping and inhibition rules.

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster']

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://webhook:5001/'
```

### Monitoring the Observability Stack

#### Stack Health Dashboard
```json
{
  "dashboard": {
    "title": "Observability Stack Health",
    "panels": [
      {
        "title": "Prometheus Uptime",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"prometheus\"}",
            "refId": "A"
          }
        ]
      },
      {
        "title": "Loki Ingestion Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(loki_distributor_samples_received_total[5m])",
            "refId": "A"
          }
        ]
      },
      {
        "title": "Tempo Spans Processed",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(tempo_distributor_spans_received_total[5m])",
            "refId": "A"
          }
        ]
      }
    ]
  }
}
```

### Cost Optimization Strategies

#### 1. Log Sampling
```yaml
# Promtail sampling
pipeline_stages:
  - match:
      selector: '{level="debug"}'
      stages:
        - drop:
            drop_counter_reason: "debug_logs"
```

#### 2. Metrics Aggregation
```yaml
# Recording rules for aggregation
groups:
  - name: aggregation
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

#### 3. Retention Policies
```yaml
# Tiered retention
retention_period: 30d
retention_size: 50GB
max_streams_per_user: 10000
```

### Troubleshooting Common Issues

#### 1. High Memory Usage
```bash
# Check memory usage
kubectl top pods -n monitoring

# Adjust resource limits
kubectl patch deployment prometheus -n monitoring -p '{"spec":{"template":{"spec":{"containers":[{"name":"prometheus","resources":{"limits":{"memory":"4Gi"}}}]}}}}'
```

#### 2. Slow Queries
```bash
# Check query performance
curl -G 'http://prometheus:9090/api/v1/query' --data-urlencode 'query=up'

# Use query optimization
# Instead of: rate(http_requests_total[1h])
# Use: rate(http_requests_total[5m])
```

#### 3. Storage Issues
```bash
# Check disk usage
df -h /var/lib/prometheus

# Clean up old data
promtool tsdb clean --retention.time=30d /var/lib/prometheus
```

### Best Practices Summary

1. **Start Small**: Begin with basic metrics and logs, add tracing later
2. **Plan Storage**: Calculate retention needs and costs upfront
3. **Use Labels Wisely**: Avoid high-cardinality labels
4. **Set Resource Limits**: Prevent resource exhaustion
5. **Monitor the Stack**: Keep an eye on your observability tools
6. **Document Everything**: Maintain runbooks and playbooks
7. **Test Alerts**: Regularly test alerting mechanisms
8. **Optimize Queries**: Use recording rules for expensive queries

[back](../)
