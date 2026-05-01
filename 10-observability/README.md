# 项目10：全链路可观测性平台

> **云平台**: 阿里云 + AWS | **难度**: ⭐⭐⭐⭐⭐ | **工具**: SLS, ARMS, CloudWatch, OpenTelemetry, Grafana  
> **场景**: 微服务架构下快速定位故障，需要统一 Metrics/Logging/Tracing 三大支柱

---

## 📖 项目概述

平台有 50+ 微服务分布在多集群+多云。故障时运维需要在 5 分钟内定位根因。构建全链路可观测平台实现 Metrics(指标) + Logging(日志) + Tracing(链路追踪) 三合一。

### 学习目标
- ✅ OpenTelemetry 标准采集方案
- ✅ SLS 日志服务 + 分析查询
- ✅ ARMS APM 全链路追踪
- ✅ Prometheus + Grafana 指标可视化
- ✅ 告警体系建设 (分级通知+群组管理)

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                     全链路可观测性平台架构                              │
│                                                                      │
│  应用层    ┌────────────────────────────────────────────┐            │
│           │  Pod / ECS                                  │            │
│           │  ┌──────────────────────────────────┐      │            │
│           │  │  OpenTelemetry Agent (Sidecar)   │      │            │
│           │  │  ├─ Traces → OTEL Collector      │      │            │
│           │  │  ├─ Metrics → Prometheus Exporter │      │            │
│           │  │  └─ Logs → Filebeat/SLS Agent    │      │            │
│           │  └──────────────────────────────────┘      │            │
│           └──────────────────┬─────────────────────────┘            │
│                              │                                       │
│  采集层    ┌──────────────────┴─────────────────────────┐            │
│           │            OpenTelemetry Collector           │            │
│           │    (DaemonSet / Deployment, 多级级联)        │            │
│           └──┬──────────────┬──────────────┬────────────┘            │
│              │              │              │                         │
│  存储层    ┌─┴──────┐  ┌───┴──────┐  ┌───┴──────────┐               │
│           │  SLS    │  │ ARMS     │  │ Prometheus    │               │
│           │ (日志)  │  │ (链路)   │  │ (指标)        │               │
│           └──┬──────┘  └──┬───────┘  └───┬───────────┘               │
│              │            │              │                            │
│  可视化    ┌─┴────────────┴──────────────┴───────────┐               │
│           │             Grafana 统一大盘               │               │
│           │  同一面板: 日志+Trace+Metrics 联动         │               │
│           └────────────────────┬─────────────────────┘               │
│                                │                                     │
│  告警层    ┌────────────────────┴─────────────────────┐               │
│           │       告警管理体系                         │               │
│           │  SLS Alert → 钉钉/飞书/企业微信/Slack      │               │
│           │  ARMS Alert → PagerDuty/Opsgenie          │               │
│           │  Prometheus Alert → Alertmanager → Webhook │               │
│           └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: OpenTelemetry 统一采集

```yaml
# OpenTelemetry Collector 配置 (Helm)
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: observability
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
      memory_limiter:
        check_interval: 5s
        limit_mib: 512
      # 资源属性丰富
      resource:
        attributes:
          - key: cloud.platform
            value: "aliyun_ack"
            action: upsert
    
    # 多 Exporters: 同时输出到多个后端
    exporters:
      # 1. ARMS (阿里云 APM, Tracing)
      alibabacloud_arms:
        endpoint: "cn-hangzhou.arms.aliyuncs.com"
        access_key_id: ${ALIYUN_AK}
        access_key_secret: ${ALIYUN_SK}
      
      # 2. SLS (日志服务, Logging + Metrics)
      alibabacloud_logservice:
        endpoint: "cn-hangzhou.log.aliyuncs.com"
        project: "ecommerce-logs"
        logstore: "traces"
        access_key_id: ${ALIYUN_AK}
        access_key_secret: ${ALIYUN_SK}
      
      # 3. Prometheus (指标)
      prometheus:
        endpoint: "0.0.0.0:8889"
        resource_to_telemetry_conversion:
          enabled: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, memory_limiter, resource]
          exporters: [alibabacloud_arms, alibabacloud_logservice]
        metrics:
          receivers: [otlp]
          processors: [batch, memory_limiter, resource]
          exporters: [prometheus, alibabacloud_logservice]
```

### Step 2: Java 应用集成 OpenTelemetry

```yaml
# Deployment 配置: OpenTelemetry Java Agent 自动注入
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        # 自动注入 OTel Java Agent
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
      - name: order-service
        image: registry.cn-hangzhou.aliyuncs.com/order:v2.5
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector.observability:4317"
        - name: OTEL_SERVICE_NAME
          value: "order-service"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "deployment.environment=prod,cloud.provider=aliyun"
        - name: OTEL_PROPAGATORS
          value: "tracecontext,baggage,b3"
```

```java
// 业务代码零侵入: 只需启动参数
// java -javaagent:opentelemetry-javaagent.jar \
//   -Dotel.service.name=order-service \
//   -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
//   -jar order-service.jar

// 手动 Span (可选)
@RestController
public class OrderController {
    @Autowired
    private Tracer tracer;
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderDTO dto) {
        Span span = tracer.spanBuilder("create-order")
            .setAttribute("order.user_id", dto.getUserId())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            // 业务逻辑
            return orderService.create(dto);
        } finally {
            span.end();
        }
    }
}
```

### Step 3: SLS 日志分析

```sql
-- SLS 日志查询 (类 SQL)
-- 常用诊断语句

-- 1. 错误分布 (最近1小时)
* | SELECT 
    service, 
    status_code, 
    COUNT(*) as error_count 
  WHERE status_code >= 500 
  GROUP BY service, status_code 
  ORDER BY error_count DESC

-- 2. P50/P95/P99 延迟 (最近30分钟)
* | SELECT 
    service,
    approx_percentile(duration, 0.50) as p50,
    approx_percentile(duration, 0.95) as p95,
    approx_percentile(duration, 0.99) as p99
  GROUP BY service

-- 3. 关联日志和 Trace: 通过 trace_id 串联调用链
* | SELECT 
    trace_id,
    service,
    span_name, 
    duration,
    status_code
  WHERE trace_id = 'xxx-yyy-zzz'
  ORDER BY timestamp

-- 4. 异常检测 (突增)
* | SELECT 
    DATE_FORMAT(__time__, '%Y-%m-%d %H:%i:00') as time_bucket,
    COUNT(*) / MAX(COUNT(*)) OVER (ORDER BY DATE_FORMAT(...)) as ratio
  GROUP BY time_bucket
```

### Step 4: Grafana 统一大盘

```yaml
# Grafana Dashboard 数据源配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.monitoring:9090
        access: proxy
        isDefault: true
      
      - name: SLS-Log
        type: aliyun-sls-datasource
        url: https://cn-hangzhou.log.aliyuncs.com
        jsonData:
          endpoint: cn-hangzhou.log.aliyuncs.com
          project: ecommerce-logs
      
      - name: ARMS-Trace
        type: aliyun-arms-datasource
        url: https://cn-hangzhou.arms.aliyuncs.com
```

```json
{
  "dashboard": {
    "title": "电商平台 - 服务全景监控",
    "panels": [
      {
        "title": "QPS (按服务)",
        "targets": [
          { "expr": "sum(rate(http_requests_total[1m])) by (service)" }
        ]
      },
      {
        "title": "错误率",
        "targets": [
          { "expr": "sum(rate(http_requests_total{status=~\"5..\"}[1m])) / sum(rate(http_requests_total[1m]))" }
        ],
        "alert": {
          "conditions": [
            { "evaluator": { "params": [0.01], "type": "gt" } }
          ]
        }
      },
      {
        "title": "P95 延迟",
        "targets": [
          { "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_bucket[1m])) by (le, service))" }
        ]
      },
      {
        "title": "SLS 日志面板 (错误日志)",
        "type": "logs",
        "targets": [
          { 
            "datasource": "SLS-Log",
            "query": "* | WHERE level='ERROR' | LIMIT 100"
          }
        ]
      }
    ]
  }
}
```

### Step 5: 分级告警体系

```yaml
# AlertManager 配置
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    # P0: 核心服务不可用 → 电话 + 钉钉
    - match:
        severity: critical
      receiver: 'on-call'
      continue: true
    # P1: 错误率 >5% → 钉钉群
    - match:
        severity: warning
      receiver: 'dingtalk-ops'
    # P2: 资源 >80% → 静默通知
    - match:
        severity: info
      receiver: 'dingtalk-silent'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://alert-gateway:8080/webhook'
  
  - name: 'on-call'
    # PagerDuty / 钉钉电话
    pagerduty_configs:
      - routing_key: ${PD_KEY}
    webhook_configs:
      - url: 'https://oapi.dingtalk.com/robot/send?access_token=ALERT_URGENT'
  
  - name: 'dingtalk-ops'
    # 钉钉运维群
    webhook_configs:
      - url: 'https://oapi.dingtalk.com/robot/send?access_token=OPS_GROUP'
        send_resolved: true
  
  - name: 'dingtalk-silent'
    webhook_configs:
      - url: 'https://oapi.dingtalk.com/robot/send?access_token=SILENT_GROUP'

# PrometheusRule: 告警规则
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sre-alerts
spec:
  groups:
    - name: service-health
      rules:
        # P0: Pod 频繁重启
        - alert: HighPodRestarts
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0.05
          for: 5m
          labels:
            severity: critical
            team: sre
          annotations:
            summary: "Pod {{ $labels.pod }} 频繁重启"
            description: "最近15分钟重启率 {{ $value }}/秒"
        
        # P1: 错误率 > 5%
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            / sum(rate(http_requests_total[5m])) by (service) > 0.05
          for: 3m
          labels:
            severity: warning
          annotations:
            summary: "{{ $labels.service }} 错误率 > 5%"
        
        # P2: 节点磁盘 > 85%
        - alert: NodeDiskPressure
          expr: |
            (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15
          for: 10m
          labels:
            severity: info
          annotations:
            summary: "Node {{ $labels.node }} 磁盘使用率 > 85%"
```

---

## 📊 常见排障流程

```
故障发现: 告警/用户反馈
    │
    ▼
1. Grafana 大盘快速确认影响范围 (QPS/Error/Latency)
    │
    ▼
2. 锁定故障服务 → SLS 查看错误日志 (trace_id)
    │
    ▼
3. ARMS 链路追踪 → 查看调用链 (哪一步慢/出错)
    │
    ▼
4. Prometheus 查看资源指标 (CPU/Memory/Network)
    │
    ▼
5. 定位根因 → 修复 → 复盘 → 更新告警规则
```

---

## 💰 成本估算

| 服务 | 月费 |
|------|------|
| SLS (500GB/天, 保留30天) | ~¥5,000 |
| ARMS APM (50节点) | ~¥3,500 |
| ARMS Prometheus (50节点) | ~¥1,600 |
| Grafana 托管 | ~¥300 |
| **合计** | **~¥10,400** |

---

## 📝 面试常见问题

**Q1: 可观测性三大支柱是什么？**
> Metrics(指标, 知道"有什么问题") + Logging(日志, 知道"为什么") + Tracing(链路追踪, 知道"哪里有问题")。三者通过 trace_id/span_id 关联。

**Q2: OpenTelemetry 解决了什么问题？**
> 统一采集标准，一份代码采集后输出到不同后端。避免被某厂商锁定 (Vendor Lock-in)。

**Q3: 如何减少告警风暴？**
> 分组 (group_by)、抑制 (inhibition_rules)、聚合窗口 (group_interval)、静默 (silence)。
