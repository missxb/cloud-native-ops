# 项目二十一: 全链路可观测性体系建设 (OpenTelemetry + SLS)

## 一、项目背景

微服务架构中应用不可见的问题：
- **故障定位困难**: 请求跨 10+ 个微服务，出问题时无法快速定位根因
- **指标/日志/追踪割裂**: 三个系统独立管理，缺少关联分析能力
- **性能瓶颈不透明**: 哪个接口响应慢？哪条 SQL 拖后腿？缺乏数据支撑
- **业务洞察缺失**: 用户下单转化率在哪一步流失？无埋点无法回答

可观测性三位一体：

```
┌──────────────┬───────────────────┬──────────────────┐
│   Metrics    │      Logs         │     Traces       │
│  (是什么)     │   (发生了什么)     │   (从哪里来)      │
│              │                   │                  │
│ Prometheus   │   Loki/Loki       │   Jaeger/Zipkin  │
│ Grafana      │   ELK             │   Tempo          │
│ AlertManager │   CloudWatch Log  │   APM            │
│ InfluxDB     │   Elasticsearch   │   SkyWalking     │
└──────────────┴───────────────────┴──────────────────┘
         ↕ OpenTelemetry Collector (统一采集、处理、分发) ↕
┌──────────────────────────────────────────────────────────────┐
│                   OTel SDK / Auto-Instrumentation           │
│                    (Java/Python/Go/NodeJS/.NET)              │
└──────────────────────────────────────────────────────────────┘
```

## 二、架构设计

### 2.1 可观测性全景图

```
┌──────────────────────────────────────────────────────────────┐
│                     Applications (微服务集群)                 │
│                                                              │
│  Product-Svc ─► Order-Svc ─► Payment-Svc ─► User-Svc        │
│  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐       │
│  │OTel SDK│    │OTel SDK│    │OTel SDK│    │OTel SDK│       │
│  └───┬────┘    └───┬────┘    └───┬────┘    └───┬────┘       │
│      ▼             ▼             ▼             ▼             │
├──────┼─────────────┼─────────────┼─────────────┼─────────────┤
│      ▼             ▼             ▼             ▼             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │         OpenTelemetry Collector (Sidecar模式)         │    │
│  │                                                       │    │
│  │  ┌─────────┐  ┌─────────┐  ┌────────────────────┐   │    │
│  │  │Metrics  │  │ Logs    │  │ Trace              │   │    │
│  │  │Receiver │  │Receiver │  │ Processor          │   │    │
│  │  │Prom/OTLP│  │ OTLP    │  │ Batch/Resource     │   │    │
│  │  └────┬────┘  └────┬────┘  └────────┬───────────┘   │    │
│  └───────┼─────────────┼────────────────┼───────────────┘    │
│          │             │                │                     │
│  ┌───────▼──┐   ┌─────▼────┐    ┌─────▼──────────┐          │
│  │ Prometh  │   │ Fluentd  │    │  Alibaba SLS   │          │
│  │ Grafana  │   │ Filebeat │    │ (阿里云日志服务)│          │
│  │ Push/G   │   │ K8s FD   │    │ ARMS APM       │          │
│  └──────────┘   └──────────┘    └────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 各云平台方案对比

| 维度 | 自建 OTel | 阿里云 SLS | AWS X-Ray | 华为云 AOM | 腾讯云 APM |
|------|----------|-----------|-----------|------------|-----------|
| Trace 支持 | ✅ 原生 | ✅ 兼容 | ✅ 自有格式 | ✅ | ✅ |
| Metrics 集成 | ✅ Prometheus | ✅ 时序库 | ❌ | ✅ | ✅ |
| Log 聚合 | ✅ Loki/ES | ✅ 核心能力 | ❌ | ✅ | ✅ |
| 自动化插桩 | ✅ SDK | ✅ 无侵入探针 | ✅ Agent | ✅ Agent | ✅ Agent |
| 自定义 Dashboard | Grafana | SLS仪表盘 | CloudWatch | Grafana | - |

## 三、核心部署方案

### 3.1 OpenTelemetry Collector 部署

```yaml
# otel-collector-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: "0.0.0.0:4317"
          http:
            endpoint: "0.0.0.0:4318"
      prometheus:
        config:
          scrape_configs:
          - job_name: 'kubernetes-pods'
            kubernetes_sd_configs:
            - role: pod
            relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: true
            
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
      resource:
        attributes:
        - key: cloud.provider
          value: aliyun
          action: upsert
        - key: k8s.pod.name
          from_attribute: k8s.pod.name
          action: upsert
      transform:
        trace_statements:
        - context: span
          statements:
          - replace_match("GET /api/*") where name matches "^GET .*$"
      
    exporters:
      # 阿里云 SLS (Traces)
      alibabacloud_sls:
        endpoint: "https://apm-cn-hangzhou.aliyuncs.com"
        access_key_id: "${ALIYUN_ACCESS_KEY_ID}"
        access_key_secret: "${ALIYUN_ACCESS_KEY_SECRET}"
        project: production-traces
        logstore: otel-traces
        
      # Prometheus (Metrics)
      prometheusremotewrite:
        endpoint: "http://prometheus-server.monitoring.svc:9090/api/v1/write"
        
      # Loki (Logs)
      loki:
        endpoint: "http://loki-gateway.logging.svc:3100/loki/api/v1/push"
        tenant_id: production
        
      # 本地调试输出
      logging:
        loglevel: debug
      
    service:
      pipelines:
        metrics:
          receivers: [otlp, prometheus]
          processors: [batch, transform]
          exporters: [prometheusremotewrite, logging]
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [alibabacloud_sls, logging]
        logs:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [loki, logging]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
      annotations:
        sidecar.opentelemetry.io/inject: "false"   # 不注入自身
    spec:
      serviceAccountName: otel-collector-sa
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.95.0
        args: ["--config", "/conf/collector-config.yaml"]
        volumeMounts:
        - name: config-volume
          mountPath: /conf
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        ports:
        - containerPort: 4317   # OTLP gRPC
        - containerPort: 4318   # OTLP HTTP
        - containerPort: 8888   # Prometheus metrics
      volumes:
      - name: config-volume
        configMap:
          name: otel-collector-config
```

### 3.2 应用侧 OTel SDK 接入

```python
# Python FastAPI + OpenTelemetry 示例
# app/main.py

from fastapi import FastAPI, Request
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.semconv.resource import ResourceAttributes
import os

# 初始化资源
resource = Resource.create({
    ResourceAttributes.SERVICE_NAME: "product-api",
    ResourceAttributes.SERVICE_VERSION: "1.2.0",
    ResourceAttributes.DEPLOYMENT_ENVIRONMENT: os.getenv("ENV", "prod"),
    ResourceAttributes.CLOUD_PROVIDER: "aliyun",
})

# 配置 Tracer Provider
trace_provider = TracerProvider(resource=resource)
trace_provider.add_span_processor(
    trace.get_tracer_provider().get_tracer("product-service")
    .start_span("init").end()
)

# OTLP Exporter → Collector
span_exporter = OTLPSpanExporter(
    endpoint="otel-collector:4317",  # Service discovery
    insecure=True
)
trace_provider.add_span_processor(
    trace.exporter.SimpleSpanProcessor(span_exporter)
)
trace.set_tracer_provider(trace_provider)

# 配置 Meter
meter_provider = metrics.MeterProvider()
metrics.set_meter_provider(meter_provider)

app = FastAPI(title="Product API")

# 自动 instrumentation
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()

@app.on_event("startup")
async def startup():
    tracer = trace.get_tracer(__name__)
    with tracer.start_as_current_span("product_api_startup"):
        print(f"Product API starting on {os.getenv('HOST', '0.0.0.0')}:{os.getenv('PORT', '8080')}")

@app.get("/api/products/{product_id}")
async def get_product(product_id: int):
    tracer = trace.get_tracer(__name__)
    with tracer.start_as_current_span("get_product") as span:
        span.set_attribute("product.id", product_id)
        span.set_attribute("request.method", "GET")
        span.set_attribute("request.path", f"/api/products/{product_id}")
        
        # DB operation tracking
        with tracer.start_as_current_span("db.query") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.operation", "SELECT")
            db_span.set_attribute("db.statement", f"SELECT * FROM products WHERE id={product_id}")
            result = await fetch_product_from_db(product_id)
        
        return {"id": result["id"], "name": result["name"]}

@app.get("/health")
async def health_check():
    return {"status": "healthy", "version": "1.2.0"}
```

```yaml
# Java Spring Boot - pom.xml 依赖
<!-- pom.xml -->
<dependencies>
    <!-- OpenTelemetry Spring Boot Starter -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
        <version>1.32.0</version>
    </dependency>
    
    <!-- Auto-instrumentation agents -->
    <dependency>
        <groupId>io.opentelemetry.javaagent</groupId>
        <artifactId>opentelemetry-javaagent</artifactId>
        <version>1.32.0</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>

<!-- application.yml -->
# application.yml
management:
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
    metrics:
      export:
        prometheus:
          enabled: true
          port: 9090
otel:
  javaagent:
    enabled: true
    config:
      exporters: otlp
```

### 3.3 Prometheus Metrics & Alert Rules

```yaml
# prometheus-rules.yaml
groups:
- name: microservice-alerts
  rules:
  # ==================== 高错误率告警 ====================
  - alert: HighErrorRate
    expr: >
      sum(rate(http_requests_total{status=~"5.."}[5m])) 
      / sum(rate(http_requests_total[5m])) > 0.05
    for: 2m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "High error rate detected in {{ $labels.service }}"
      description: "Error rate is {{ humanize $value | multiply 100 }}% (threshold: 5%)"
  
  # ==================== 高延迟告警 ====================
  - alert: HighLatencyP99
    expr: >
      histogram_quantile(0.99, 
        sum(rate(http_request_duration_seconds_bucket[5m])) 
        by (le, service)
      ) > 2.0
    for: 3m
    labels:
      severity: warning
      team: backend
    annotations:
      summary: "P99 latency > 2s for {{ $labels.service }}"
  
  # ==================== 服务不可用 ====================
  - alert: ServiceDown
    expr: up{job="microservices"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "{{ $labels.instance }} is down!"
  
  # ==================== Pod频繁重启 ====================
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} crash looping (restarts: {{ $value }})"

- name: infrastructure-alerts
  rules:
  - alert: NodeDiskPressure
    expr: kube_node_status_condition{condition="DiskPressure",status_true="true"} == 1
    labels:
      severity: critical
  
  - alert: ContainerOOMKilled
    expr: increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[1h]) > 0
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.container }} was OOM killed"
```

### 3.4 Grafana Dashboard JSON (简化版)

```json
{
  "dashboard": {
    "title": "Microservices Overview",
    "templating": {
      "list": [{
        "name": "namespace",
        "type": "query",
        "datasource": "prometheus",
        "query": "label_values(namespace)"
      }]
    },
    "panels": [
      {
        "title": "Request Rate (QPS)",
        "type": "timeseries",
        "targets": [{
          "expr": "sum(rate(http_requests_total{namespace=\"$namespace\"}[5m])) by (service)",
          "legendFormat": "{{service}}"
        }]
      },
      {
        "title": "Error Rate (%)",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_requests_total{namespace=\"$namespace\", status=~\"5..\"}[5m])) / sum(rate(http_requests_total{namespace=\"$namespace\"}[5m])) * 100"
        }]
      },
      {
        "title": "P50/P95/P99 Latency",
        "type": "timeseries",
        "targets": [{
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{namespace=\"$namespace\"}[5m])) by (le, service))",
          "legendFormat": "{{service}} - P50"
        }, {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace=\"$namespace\"}[5m])) by (le, service))",
          "legendFormat": "{{service}} - P95"
        }]
      }
    ]
  }
}
```

## 四、Trace ID 全链路透传

```yaml
# istio-telemetry-propagation.yaml
# Istio 确保 Trace Header (W3C Trace Context) 透传到下游服务
apiVersion: networking.istio.io/v1beta1
kind: EnvoyFilter
metadata:
  name: propagate-trace-context
  namespace: app-prod
spec:
  workloadSelector:
    labels:
      app: product-service
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
          inlineCode: |
            function envoy_on_response(response_handle)
              local traceparent = response_handle:headers():get("x-b3-traceparent")
              if traceparent ~= "" then
                response_handle:headers():add("traceparent", traceparent)
              end
            end
```

```python
# Python: 手动关联父 Trace 到子请求
import requests
from opentelemetry import trace
from opentelemetry.propagators.textmap import DictGetter

tracer = trace.get_tracer(__name__)

def call_upstream_service(url: str) -> dict:
    """调用下游服务时自动注入 Trace Context"""
    carrier = {}
    # 获取当前 Span 的 propagation context
    from opentelemetry import context
    
    with tracer.start_as_current_span("call-upstream") as span:
        # OTel 自动将 traceparent/baggage 注入 header
        headers = {}
        from opentelemetry.propagate import inject
        inject(carrier=headers, setter=requests.HEADERS_SETTER)
        
        response = requests.get(url, headers=headers)
        span.set_attribute("http.response.status_code", response.status_code)
        return response.json()
```

## 五、成本估算

| 组件 | 规格 | 月费用 |
|------|------|--------|
| OTel Collector (DaemonSet) | 每节点 ~100m CPU | 自付 K8s 资源 |
| Prometheus | 2xC8G + 200GB | ¥1,200/月 |
| Grafana | 2xC4G | ¥400/月 |
| Loki + MinIO | 存储 ~500GB | ¥800/月 |
| Jaeger | 2xC4G | ¥400/月 |
| 阿里云 SLS (APM) | 企业版 | ¥3,000/月 |

## 六、面试考点

1. **Trace vs Span vs Segment**: Trace 是完整请求链路，Span 是链路上的一个操作单元
2. **OTLP 协议**: OpenTelemetry 标准协议，gRPC/HTTP 两种传输方式
3. **采样策略**: 头部采样(deterministic) vs 尾部采样(final) vs 自适应采样
4. **B3 vs W3C Trace Context**: W3C 是标准，B3 是 Zipkin 私有格式，Istio 默认支持两者
5. **Service Map**: 从 Trace 数据构建拓扑图，展示服务间调用关系

## 七、课后练习

1. 部署 OTel Collector + Prometheus + Grafana + Jaeger 完整栈
2. 在 FastAPI 应用中接入 OTel SDK 并验证 Trace 传递
3. 编写 5 条关键业务告警规则并在 Grafana 中设置
4. 使用 Grafana Explore 功能按 Trace ID 关联查询对应时间戳的 Logs
