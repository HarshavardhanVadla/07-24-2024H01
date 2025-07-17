# Prometheus Data Storage: Best Practices for Long-Term Retention

## Executive Summary

This document outlines the best practices for storing all data collected by Prometheus from monitored instances. While Prometheus excels at real-time monitoring, its default 15-day retention period is insufficient for long-term analysis, compliance requirements, and trend identification. This guide provides comprehensive solutions for extending data retention efficiently and cost-effectively.

## Understanding Prometheus Storage Limitations

### Default Constraints
- **Default Retention**: 15 days
- **Storage Type**: Local disk only
- **Scalability**: Limited by single-server capacity
- **Query Performance**: Degrades with increased retention on local storage

### Why Extended Storage Matters
- **Trend Analysis**: Identify gradual performance changes over months/years
- **Compliance**: Meet regulatory requirements (finance, healthcare, government)
- **Incident Investigation**: Access historical context for complex debugging
- **Capacity Planning**: Make informed infrastructure decisions based on long-term data
- **Seasonal Patterns**: Understand yearly usage cycles and traffic patterns

## Storage Strategy Options

### 1. Extended Local Storage (Short to Medium Term)

#### Configuration
```bash
# Time-based retention
--storage.tsdb.retention.time=90d

# Size-based retention
--storage.tsdb.retention.size=100GB

# Combined approach (recommended)
--storage.tsdb.retention.time=90d
--storage.tsdb.retention.size=100GB
```

#### Implementation Examples

**SystemD Service:**
```bash
# Edit service file
sudo vi /etc/systemd/system/prometheus.service

# Add retention flags
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=90d \
  --storage.tsdb.retention.size=100GB

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart prometheus
```

**Docker Compose:**
```yaml
services:
  prometheus:
    image: prom/prometheus:v2.48.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d'
      - '--storage.tsdb.retention.size=100GB'
      - '--storage.tsdb.wal-compression'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
```

**Kubernetes:**
```yaml
spec:
  containers:
  - name: prometheus
    args:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d'
      - '--storage.tsdb.retention.size=100GB'
      - '--storage.tsdb.wal-compression'
```

#### Storage Estimation
```
Daily Storage = Ingestion Rate × 24 hours × Bytes per Sample
Example: 10,000 samples/sec × 86,400 seconds × 2 bytes = ~1.7 GB/day
For 90 days: 1.7 GB × 90 = ~153 GB storage needed
```

### 2. Remote Storage Solutions (Long-Term)

#### A. Prometheus Remote Write Protocol

**Configuration in prometheus.yml:**
```yaml
remote_write:
  - url: "http://remote-storage-endpoint:8080/api/v1/write"
    queue_config:
      capacity: 10000
      max_shards: 1000
      min_shards: 1
      max_samples_per_send: 1000
      batch_send_deadline: 5s
```

#### B. Popular Remote Storage Options

**1. Thanos (Recommended for Large Scale)**
```yaml
# thanos-sidecar.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-sidecar
spec:
  template:
    spec:
      containers:
      - name: thanos-sidecar
        image: thanosio/thanos:v0.32.0
        args:
          - sidecar
          - --tsdb.path=/prometheus
          - --prometheus.url=http://localhost:9090
          - --objstore.config-file=/etc/thanos/objstore.yml
        volumeMounts:
        - name: objstore-config
          mountPath: /etc/thanos
```

**Object Storage Configuration:**
```yaml
# objstore.yml
type: S3
config:
  bucket: "prometheus-metrics"
  endpoint: "s3.amazonaws.com"
  access_key: "YOUR_ACCESS_KEY"
  secret_key: "YOUR_SECRET_KEY"
  region: "us-east-1"
```

**2. Cortex/Mimir (Multi-Tenant)**
```yaml
# cortex-config.yml
server:
  http_listen_port: 8080
  grpc_listen_port: 9095

distributor:
  ring:
    kvstore:
      store: consul
      consul:
        host: consul:8500

ingester:
  lifecycler:
    ring:
      kvstore:
        store: consul
        consul:
          host: consul:8500

storage:
  engine: blocks
  
blocks_storage:
  backend: s3
  s3:
    endpoint: s3.amazonaws.com
    bucket_name: cortex-metrics
    access_key_id: YOUR_ACCESS_KEY
    secret_access_key: YOUR_SECRET_KEY
```

**3. VictoriaMetrics (Cost-Effective)**
```yaml
# docker-compose.yml
version: '3.8'
services:
  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    ports:
      - "8428:8428"
    volumes:
      - vmdata:/victoria-metrics-data
    command:
      - '--storageDataPath=/victoria-metrics-data'
      - '--retentionPeriod=12'  # 12 months
      - '--httpListenAddr=:8428'
```

**4. GreptimeDB (Efficient Alternative)**
```yaml
# Significantly lower resource usage (80-90% reduction)
# Native Prometheus compatibility
# Unified observability (metrics, logs, traces)
services:
  greptimedb:
    image: greptime/greptimedb:latest
    ports:
      - "4000:4000"
      - "4001:4001"
      - "4002:4002"
    environment:
      - GREPTIMEDB_DATANODE__STORAGE__DATA_HOME=/data
    volumes:
      - greptimedb_data:/data
```

### 3. Hybrid Approach (Recommended)

#### Tiered Storage Strategy
```
Hot Tier (Prometheus):  7-30 days  - Fast queries, full resolution
Warm Tier (Remote):     1-12 months - Good performance, full resolution  
Cold Tier (Archive):    1+ years    - Slower queries, downsampled data
```

#### Implementation Example
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "recording_rules.yml"

remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      capacity: 10000
      max_samples_per_send: 1000

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

```yaml
# recording_rules.yml - Pre-aggregate for long-term storage
groups:
  - name: long_term_metrics
    interval: 5m
    rules:
      - record: node:cpu_utilization:avg5m
        expr: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      
      - record: node:memory_utilization:avg5m
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
      
      - record: node:disk_utilization:avg5m
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

## Storage Optimization Techniques

### 1. Data Compression
```bash
# Enable WAL compression
--storage.tsdb.wal-compression

# This can reduce storage by 50-70%
```

### 2. Recording Rules for Downsampling
```yaml
# Create aggregated metrics for long-term storage
groups:
  - name: downsampling
    interval: 1m
    rules:
      - record: node:cpu:avg1m
        expr: avg(rate(node_cpu_seconds_total[1m])) by (instance)
      
      - record: node:memory:avg1m  
        expr: avg(node_memory_MemAvailable_bytes) by (instance)
```

### 3. Metric Filtering
```yaml
# Only store essential metrics long-term
remote_write:
  - url: "http://remote-storage:8080/api/v1/write"
    write_relabel_configs:
      - source_labels: [__name__]
        regex: 'node_cpu_seconds_total|node_memory_MemAvailable_bytes|node_filesystem_avail_bytes'
        action: keep
```

## Cost Optimization Strategies

### 1. Resource Efficiency Comparison
| Solution | CPU Usage | Memory Usage | Cost (10M series/year) |
|----------|-----------|--------------|----------------------|
| Thanos   | High      | 100-120 GiB | $15,000 USD         |
| Cortex   | High      | 80-100 GiB  | $12,000 USD         |
| VictoriaMetrics | Medium | 40-60 GiB | $6,000 USD |
| GreptimeDB | Low    | 10-20 GiB   | $2,000 USD          |

### 2. Storage Cost Optimization
```bash
# S3 lifecycle policies for automated tiering
{
  "Rules": [
    {
      "ID": "prometheus-metrics-lifecycle",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
```

## Monitoring and Alerting

### Key Metrics to Monitor
```yaml
# Prometheus storage metrics
- prometheus_tsdb_storage_blocks_bytes_total
- prometheus_tsdb_head_series
- prometheus_tsdb_head_chunks
- prometheus_local_storage_memory_chunks
- prometheus_tsdb_compactions_total

# Alerting rules
groups:
  - name: storage_alerts
    rules:
      - alert: PrometheusStorageSpaceRunningOut
        expr: predict_linear(prometheus_tsdb_storage_blocks_bytes[1h], 24 * 3600) / prometheus_tsdb_storage_blocks_bytes_total > 0.8
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Prometheus storage space running out"
          description: "Prometheus storage is predicted to run out in 24 hours"
```

## Implementation Roadmap

### Phase 1: Immediate (1-2 weeks)
1. **Extend Local Retention**: Increase to 30-90 days
2. **Enable Compression**: Add WAL compression flag
3. **Storage Monitoring**: Implement disk usage alerts
4. **Backup Strategy**: Set up regular data backups

### Phase 2: Short-term (1-2 months)  
1. **Recording Rules**: Create aggregated metrics
2. **Remote Write**: Implement basic remote storage
3. **Query Optimization**: Optimize frequently used queries
4. **Documentation**: Create operational runbooks

### Phase 3: Long-term (3-6 months)
1. **Full Remote Storage**: Deploy Thanos/Cortex/VictoriaMetrics
2. **Tiered Architecture**: Implement hot/warm/cold storage
3. **Automated Scaling**: Set up auto-scaling for storage
4. **Cost Optimization**: Implement lifecycle policies

## Best Practices Summary

### Do's
- ✅ Use combined time and size-based retention
- ✅ Enable compression for storage efficiency
- ✅ Implement recording rules for common queries
- ✅ Monitor storage usage and set alerts
- ✅ Use remote storage for long-term retention
- ✅ Plan for storage growth and scaling
- ✅ Document retention policies and procedures

### Don'ts
- ❌ Don't increase retention without monitoring disk usage
- ❌ Don't store all metrics long-term without filtering
- ❌ Don't ignore query performance impact
- ❌ Don't forget to backup critical data
- ❌ Don't choose storage solution without cost analysis
- ❌ Don't implement complex solutions without testing

## Conclusion

The best approach for storing all Prometheus data depends on your specific requirements:

- **Small Scale (< 1M series)**: Extended local storage with compression
- **Medium Scale (1-10M series)**: VictoriaMetrics or GreptimeDB
- **Large Scale (> 10M series)**: Thanos or Cortex with object storage
- **Cost-Sensitive**: GreptimeDB or VictoriaMetrics
- **Enterprise**: Thanos with comprehensive tiering strategy

Start with extended local storage, then gradually implement remote storage as your needs grow. Always monitor performance and costs, and choose solutions that align with your operational capabilities and budget constraints.

## Additional Resources

- [Prometheus Storage Documentation](https://prometheus.io/docs/prometheus/latest/storage/)
- [Thanos Documentation](https://thanos.io/tip/thanos/getting-started.md/)
- [VictoriaMetrics Documentation](https://docs.victoriametrics.com/)
- [GreptimeDB Documentation](https://docs.greptime.com/)
- [Cortex Documentation](https://cortexmetrics.io/docs/)