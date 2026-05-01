# 项目二十: 云原生弹性伸缩实战 (HPA/VPA/KEDA)

## 一、项目背景

微服务应用中固定资源分配的问题：
- **潮汐流量**: 白天业务高峰需要 10x 实例，凌晨仅需 30%
- **突发流量**: 秒杀活动/QC 直播导致瞬时流量暴增 100x
- **成本浪费**: 为保证峰值容量长期超配资源
- **冷启动延迟**: 水平扩缩容时需要等待 Pod 就绪

多粒度弹性策略互补：
| 维度 | HPA | VPA | Cluster Autoscaler | KEDA |
|------|-----|-----|-------------------|------|
| 控制层 | Pods 数量 | 容器资源请求 | Node 数量 | Pods 数量 |
| 指标源 | CPU/Mem | 实际使用率 | 节点资源需求 | 外部事件队列 |
| 响应速度 | ~30s | ~5min | ~10s-5min | ~秒级 |
| 适用场景 | 已知负载模式 | 优化资源利用 | 集群整体瓶颈 | 事件驱动架构 |

## 二、架构设计

### 2.1 弹性伸缩全景图

```
                        ┌─────────────────────────────┐
                        │    Metrics Server           │
                        │  (CPU/Memory aggregation)   │
                        └──────────┬──────────────────┘
                                   │ /apis/metrics.k8s.io
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────▼───────┐   ┌───────▼──────┐   ┌────────▼────────┐
     │    HPA          │   │     VPA       │   │     KEDA        │
     │ (Resource-based│   │(Resource-based│   │(Event-driven   │
     │  scaling)      │   │  scaling)    │   │  scaling)      │
     └────────┬───────┘   └───────┬──────┘   └────────┬────────┘
              │                   │                    │
     ┌────────▼───────────────────▼────────────────────▼───────┐
     │                   Deployment/StatefulSet                  │
     │         Product-Svc (3→20 pods on demand)               │
     └─────────────────────────────────────────────────────────┘
                                   │ Scale Up/Down
                           ┌───────▼────────┐
                           │ Cluster        │
                           │ Autoscaler     │
                           │ (Add/Remove    │
                           │  Nodes)        │
                           └────────────────┘
```

### 2.2 多云弹性服务对比

| 方案 | 阿里云 | AWS | 华为云 | 腾讯云 |
|------|--------|-----|--------|--------|
| POD弹性 | ACK增强HPA | EKS HPA | CCE HPA | TKE HPA |
| 节点弹性 | ACK CA | EKS CA | CCE CA | TSC CA |
| 事件伸缩 | 函数计算 FC | Lambda + SQS | FunctionGraph | SCF |
| 预测伸缩 | AHPA(阿里自研) | - | - | 弹性伸缩组 |

## 三、核心组件部署

### 3.1 HPA (Horizontal Pod Autoscaler) v2

```yaml
# hpa-v2-resource.yaml - 基于CPU和内存的横向扩展
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: app-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU 目标使用率 70%
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # 内存 目标使用率 80%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # QPS 自定义指标
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"  # 每个 pod 平均 100 QPS
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # 放大时稳定窗口
      policies:
      - type: Percent
        value: 100                        # 每次最多增加 100%
        periodSeconds: 60
      - type: Pods
        value: 4                          # 每次最多增加 4 个 pods
        periodSeconds: 60
      selectPolicy: Max                   # 取最激进的策略
    scaleDown:
      stabilizationWindowSeconds: 300    # 缩小稳定窗口更长(防抖动)
      policies:
      - type: Percent
        value: 10                         # 每次最多减少 10%
        periodSeconds: 60
      selectPolicy: Min                   # 取保守策略
---
# 基于 QPS/SLO 的自定义指标 HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-service-hpa-slo
  namespace: app-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-service
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Object
    object:
      metric:
        name: request-duration-ms
      describedObject:
        apiVersion: networking.istio.io
        kind: VirtualService
        name: checkout-vs
      target:
        type: Pods
        name: checkout-service
      targetMetricValue: 200             # P99 < 200ms
```

### 3.2 VPA (Vertical Pod Autoscaler)

```bash
#!/bin/bash
# install-vpa.sh - VPA 安装脚本
set -euo pipefail

echo "=== 安装 Vertical Pod Autoscaler ==="

# VPA 组件包括: Recommender, Updater, Admitter
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/v1.21.0/vertical-pod-autoscaler-1.21.0.tgz

# 验证
kubectl get pods -n kube-system -l app=(vpa-recommender,vpa-updater,vpa-admission)

# 注意: VPA 默认在 Init 模式下运行（仅推荐不执行）
# 切换到 Auto 模式需要在 VPA CR 中指定 updateMode
```

```yaml
# vpa-product-service.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: product-service-vpa
  namespace: app-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  updatePolicy:
    updateMode: "Initial"    # Initial=仅首次推荐, Auto=自动调整, Off=禁用
  resourcePolicy:
    containerPolicies:
    - containerName: product-api
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: "4"
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits   # 同时修改 requests 和 limits
---
# Mode: Auto - 全自动垂直弹性
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa-auto
  namespace: app-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: payment-api
      minAllowed:
        cpu: 200m
        memory: 256Mi
      maxAllowed:
        cpu: "2"
        memory: 2Gi
```

### 3.3 KEDA (Kubernetes Event-driven Autoscaling)

```yaml
# keda-scaledobject-kafka.yaml - Kafka 消费积压驱动伸缩
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-kafka-scaled
  namespace: app-prod
spec:
  scaleTargetRef:
    name: order-consumer            # 对应 Deployment name
  minReplicaCount: 2                # 最少 2 个消费者实例
  maxReplicaCount: 30               # 最多 30 个
  triggers:
  # Kafka Topic 分区消费积压
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker-1:9092,kafka-broker-2:9092
      consumerGroup: order-processors
      topic: user-orders
      lagThreshold: "1000"           # 每个分区 > 1000 条消息触发扩容
      lagCalculationTime: "5"        # 每 5 秒检查一次
      consumptionPerInterval: "100"  # 每个实例每秒处理 100 条
  # 辅助触发器: HTTP QPS
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring.svc:9090
      metricName: http_requests_total
      query: rate(http_requests_total{service="order-consumer"}[1m])
      threshold: "500"
      containerResource:
        name: cpu
        container: order-api
---
# KEDA Auth for Kafka (SASL/SCRAM)
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
  namespace: app-prod
spec:
  secretTargetRef:
  - parameter: bootstrapServers
    name: kafka-credentials
    key: bootstrapServers
  - parameter: saslUsername
    name: kafka-credentials
    key: username
  - parameter: saslPassword
    name: kafka-credentials
    key: password
  - parameter: saslMechanisms
    name: kafka-credentials
    key: mechanism
```

### 3.4 Cluster Autoscaler (节点弹性)

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        cluster-autoscaler.kubeads.org/latest-version: "1.28.0"
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/google_containers/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=alicloud                 # 或 aws/gke
        - --scale-down-delay-after-add=10m          # 缩容冷却时间
        - --scale-down-unneeded-time=10m            # 节点不满足判断周期
        - --scale-down-utilization-threshold=0.5    # 利用率低于 50% 考虑缩容
        - --max-graceful-termination-sec=600        # 安全退出最大时间
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups=true
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste                     # 最优填充策略
        - --scan-interval=10s
        env:
        - name: ALIBABA_CLOUD_REGION_ID
          valueFrom:
            configMapKeyRef:
              name: region-info
              key: regionId
---
# 配置弹性伸缩组
apiVersion: v1
kind: ConfigMap
metadata:
  name: ca-config
  namespace: kube-system
data:
  auto-discovery: |
    asg-tag: k8s.io/cluster-autoscaler/enabled=k8s.io/cluster-autoscaler/my-cluster
    balance-similar-node-groups: "true"
    skip-nodes-with-system-pods: "true"
```

### 3.5 AHPA (阿里云高级 HPA - 预测性弹性)

```yaml
# ahp-predictive-scaling.yaml
# 阿里云 ACK 特有的预测性弹性能力
apiVersion: ack.alibabacloud.com/v1alpha1
kind: AdvancedHPA
metadata:
  name: flash-sale-prevention
  namespace: app-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 100
  prediction:
    enabled: true
    model: prophet                      # Facebook Prophet 时间序列模型
    horizonSeconds: 300                 # 提前 5 分钟预测
    checkPointSeconds: 60               # 每分钟采样一次
    dataPoints: 288                     # 保留最近 4 小时数据
    algorithm: ARIMA                    # 备选算法
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  cooldown:
    scaleUpGracePeriod: 0               # 预测性扩容无冷却期
    scaleDownGracePeriod: 300
  strategy:
    predictScaleUp: true                # 允许基于预测扩容
    bufferPercentage: 10                # 预留 10% 缓冲容量
    emergencyThreshold: 90              # CPU > 90% 立即扩容
```

## 四、运维管理

### 4.1 伸缩监控仪表盘 SQL

```sql
-- Prometheus 查询: HPA 当前 vs 最大值
hpa_current_replicas / hpa_max_replicas * 100

-- 当前伸缩活跃状态
kube_hpa_status_current_replicas

-- VPA 推荐资源值
vpa_recommendation_containerresources_average_utilization

-- KEDA active count
keda_scaled_object_active_count

-- Cluster Autoscaler 空闲节点数
ca_node_groups_nodes_count * on(group) group_left(status) kube_node_status_condition{condition="Ready",status_true="true"}
```

### 4.2 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| HPA Status: "no such metric" | Metrics Server 未就绪 | `kubectl get deployment metrics-server -n kube-system` |
| HPA 不停震荡 | 伸缩窗口重叠 | 调整 scaleUp/stabilizationWindowSeconds |
| VPA 推荐但无法应用 | Deployment 缺 updateStrategy | 设置 RollingUpdate 策略 |
| KEDA Scaling to 0 | idleThreshold 配置不当 | 调高阈值确保至少保持 1 个 pod |
| Pod Pending 因为资源不足 | CA 未触发扩容 | 检查 ASG 是否有限制; 确认标签匹配 |

## 五、成本估算

| 资源 | 规格 | 月费用估算 |
|------|------|-----------|
| KEDA Operator | 免费开源 | ¥0 |
| Cluster Autoscaler | 免费开源 | 自付K8s资源 |
| AHPA (ACK Pro) | 按量计费 | ¥0.03/pod-次 |
| 弹性 ECS 节点 | c6.large (按需) | ¥0.6/h (峰值时使用) |

## 六、面试考点

1. **HPA V2 API**: 支持多指标聚合、行为策略(policy + selectPolicy)、自定义指标(Metrics API)
2. **VPA UpdateMode**: Initial(仅建议)/Auto(自动apply+滚动重启)/Off(完全禁用)
3. **HPA + VPA 冲突**: VPA 调整请求 → HPA 看使用率决定扩缩 → 可能互相矛盾；通常先用VPA优化再上HPA
4. **冷启动优化**: readinessProbe + initContainer + PreStop hook 有序终止
5. **预测性伸缩**: Prophet/ARIMA 时间序列分析历史趋势 + 节假日因子

## 七、课后练习

1. 创建 HPA 实现基于 CPU 使用率和自定义 HTTP QPS 的弹性伸缩
2. 使用 KEDA 连接 Kafka 实现消费积压驱动的自动伸缩
3. 配置 Cluster Autoscaler 让节点池根据 pod 资源请求自动增减节点
4. 编写一个压测脚本验证 HPA 从 3 扩容到 10 的实际耗时
