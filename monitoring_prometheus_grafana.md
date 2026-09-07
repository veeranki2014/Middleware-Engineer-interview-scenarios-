# Monitoring — Prometheus & Grafana for WebSphere and Kubernetes

---

## Which Monitoring Tools Are Used

Layered answer — infra-level, app-level, and visualization:

| Layer | Tool |
|---|---|
| Metrics collection | Prometheus |
| Visualization/dashboards | Grafana |
| WebSphere-specific metrics | PMI (Performance Monitoring Infrastructure) exposed via JMX, scraped using JMX Exporter |
| Kubernetes-native metrics | kube-state-metrics, cAdvisor (built into kubelet) |
| Log aggregation | ELK stack (Elasticsearch, Logstash/Fluentd, Kibana) or Splunk |
| Alerting | Alertmanager (paired with Prometheus) |
| Infra/host-level | Node Exporter |

> I use Prometheus as the central metrics store, Grafana for dashboards, and JMX Exporter to bridge WebSphere's PMI metrics into Prometheus's format since WAS doesn't expose Prometheus-native metrics out of the box. On Kubernetes, kube-state-metrics and cAdvisor cover cluster/pod-level metrics, and everything rolls up into the same Grafana instance for a unified view.

---

## How to Configure Prometheus and Grafana

### Step 1: Deploy Prometheus (Kubernetes-native, via Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

This bundles Prometheus, Alertmanager, Grafana, and Node Exporter together — the standard approach rather than installing each piece manually.

### Step 2: Configure Scrape Targets (`prometheus.yml`)

```yaml
scrape_configs:
  - job_name: 'websphere-jmx-exporter'
    static_configs:
      - targets: ['was-jmx-exporter:9404']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Step 3: Bridge WebSphere PMI Metrics into Prometheus Format

WAS doesn't natively speak Prometheus's metrics format — a **JMX Exporter** sidecar/agent bridges the gap:

```yaml
# jmx_exporter_config.yaml
rules:
  - pattern: 'WebSphere<type=JVM><>HeapSize'
    name: was_jvm_heap_size_bytes
  - pattern: 'WebSphere<type=ThreadPoolManager><>ActiveCount'
    name: was_threadpool_active_count
```

```bash
# Attach as a Java agent on JVM startup
-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx_exporter/jmx_exporter_config.yaml
```

Set this in `genericJvmArguments` under the WAS Admin Console (**Servers → Application Servers → [server] → Process Definition → Java Virtual Machine**), or via `wsadmin` for automation.

### Step 4: Deploy Grafana and Connect the Data Source

```bash
# If not bundled via Helm chart above, standalone install:
helm install grafana grafana/grafana -n monitoring
```

In Grafana UI: **Configuration → Data Sources → Add data source → Prometheus** → point to `http://prometheus-server:9090`.

### Step 5: Import/Build Dashboards

- Use community dashboard IDs from grafana.com (e.g., Node Exporter Full, Kubernetes Cluster Monitoring) for infra baseline
- Build custom dashboards for WAS-specific PMI metrics using PromQL queries against the JMX Exporter data

```promql
# Example PromQL query in a Grafana panel
rate(was_threadpool_active_count[5m])
```

### Step 6: Configure Alerting Rules

```yaml
# alert_rules.yml
groups:
  - name: was_alerts
    rules:
      - alert: HighHeapUsage
        expr: was_jvm_heap_used_bytes / was_jvm_heap_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "WAS JVM heap usage above 85%"
```

Wired to Alertmanager, which routes to Slack/PagerDuty/email.

---

## What to Monitor — WebSphere vs Kubernetes

### WebSphere-Specific Metrics (via PMI/JMX Exporter)

| Category | Metric |
|---|---|
| JVM Heap | Heap used, heap max, heap %, GC pause time, GC frequency |
| Thread Pools | Active threads, pool size, percent maxed (thread pool exhaustion is a classic prod issue) |
| Connection Pools | Active connections, pool size, wait time, timeout count |
| Servlet/EJB | Request count, response time, concurrent requests |
| Transactions | JTA transaction rate, rollback count, timeout count |
| Sessions | Active session count, session creation/invalidation rate |
| JMS | Queue depth, message processing rate (backlog is an early warning sign) |
| Server availability | Up/down status per cluster member |

### Kubernetes-Specific Metrics

| Category | Metric |
|---|---|
| Pod-level | CPU/memory usage vs requests/limits, restart count, `OOMKilled` events |
| Node-level | Node CPU/memory/disk pressure, disk usage, network I/O |
| Deployment/ReplicaSet | Desired vs available replicas, rollout status |
| Cluster health | API server latency, etcd health, scheduler/controller-manager status |
| Networking | Ingress request rate/latency, service endpoint availability |
| Storage | PV/PVC usage and capacity |
| HPA | Current vs target replica count, scaling events |
| Events | Pod evictions, `CrashLoopBackOff`, `ImagePullBackOff` |

---

## The Connecting Insight (Interview Soundbite)

> For WebSphere, most of what I'm watching maps directly to the incidents I've handled — heap and GC metrics catch OOM before it happens, thread pool metrics catch the high-CPU/thread-exhaustion scenario before it pages someone at 2am, and connection pool metrics catch a DB-side leak early. For Kubernetes, it's the same philosophy shifted up a layer — I'm watching resource requests/limits against actual usage so pods get right-sized before they hit `OOMKilled`, and watching replica/rollout health so a bad deployment gets caught by readiness probes and alerting rather than by a customer.