# 项目十六: Service Mesh 网格化治理 (Istio + ASM)

## 一、项目背景

随着微服务架构规模扩大，传统的服务发现、负载均衡、熔断限流等功能嵌入应用代码后导致：
- 业务代码与基础设施逻辑耦合严重
- 每次协议升级（HTTP/1.1 → HTTP/2/gRPC）需全量改造
- 灰度发布、A/B 测试缺少统一管控
- 跨语言技术栈难以统一治理策略

Service Mesh 通过将流量控制旁路到 Sidecar 代理，实现**业务逻辑与流量治理解耦**。

## 二、架构设计

### 2.1 整体架构图

```
                    ┌──────────────────────────────────────┐
                    │         Service Mesh Control Plane   │
                    │  ┌─────────┐  ┌──────────────────┐  │
                    │  │ Istiod  │  │  Policy/Telemetry │  │
                    │  │(XDS API)│  │     Cadvisor      │  │
                    │  └────┬────┘  └────────┬─────────┘  │
                    └───────┼────────────────┼─────────────┘
                            │ CDS/EDS/RDS/LDS │
              ┌─────────────┴─────────────────┴────────────┐
              │              Data Plane                     │
              │                                              │
    ┌─────────▼────────┐  ┌────────────────┐  ┌───────────▼───────┐
    │  Product Svc     │  │ Payment Svc    │  │ Order Svc        │
    │  ┌────────────┐  │  │ ┌────────────┐ │  │ ┌────────────┐  │
    │  │ App Pod    │  │  │ │ App Pod   │ │  │ │ App Pod   │  │
    │  └──────┬─────┘  │  │ └──────┬────┘ │  │ └──────┬────┘  │
    │  ┌──────▼─────┐  │  │ ┌──────▼────┘ │  │ ┌──────▼────┘  │
    │  │ Envoy Side │  │  │ │ Envoy Side│ │  │ │ Envoy Side │  │
    │  └────────────┘  │  │ └──────────┘ │  │ └────────────┘  │
    └──────────────────┘  └────────────────┘  └───────────────┘
             mTLS ←────────────→ gRPC/HTTP2 ────────────────→
```

### 2.2 技术选型对比

| 维度 | Istio | AWS App Mesh | Linkerd |
|------|-------|-------------|---------|
| 协议支持 | HTTP/gRPC/TCP/UDP/MongoDB | HTTP/V2/gRPC/TCP | HTTP/TCP/gRPC |
| 多集群 | ✅ 原生支持 | ❌ 需跨Account | ⚠️ 社区方案 |
| 配置方式 | YAML/Kubernetes CRD | Console/API | YAML |
| 资源消耗 | 中 (Envoy ~150MB) | 低 (AppMesh Agent) | 极低 (~5MB) |
| 生态 | CNCF毕业，最大生态 | AWS深度集成 | Burstable开源 |

## 三、核心组件部署

### 3.1 阿里云 ASM 安装

```bash
# 1. 下载 ASM CLI
curl -o asmctl https://asm-public.oss-cn-hangzhou.aliyuncs.com/asm/asmctl-linux-amd64
chmod +x asmctl

# 2. 初始化 ASM
asmctl init --cluster-id <your-cluster-id> \
  --namespace istio-system

# 3. 查看 ASM 控制台
kubectl get pods -n istio-system
# 预期输出:
# NAME                              READY   STATUS
# asm-control-plane-xxxxx           2/2     Running
# asm-gateway-xxxxx                 2/2     Running
```

### 3.2 Kubernetes 自建 Istio 安装

```bash
# 1. 下载 Istio 1.21
ISTIO_VERSION=1.21.2
curl -L https://istio.io/downloadIstio | ISTIO_VER=${ISTIO_VERSION} sh -

cd istio-${ISTIO_VERSION}

# 2. 添加 PATH
export PATH=$PWD/bin:$PATH

# 3. 安装 Istio base + default profile
istioctl install --set profile=default -y

# 4. 验证安装
kubectl get pods -n istio-system
# NAME                                    READY   STATUS
# istiod-7d99c58f94-abcde                1/1     Running
# istio-egressgateway-5f6b8c7d9-xyz      1/1     Running
# istio-ingressgateway-6c8d9e0f1-mnp     1/1     Running
```

### 3.3 Sidecar 自动注入

```yaml
# namespace-label-injection.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservice-prod
  labels:
    istio-injection: enabled   # 关键标签
---
# 或者用 istio sidecar injector mutating webhook
# 对所有新 pod 自动注入 Envoy sidecar
```

```yaml
# 带 resource limits 的 deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: microservice-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product
      version: v1
  template:
    metadata:
      labels:
        app: product
        version: v1
        sidecar.istio.io/inject: "true"
      annotations:
        # 自定义 sidecar 资源配置
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyMemory: "128Mi"
    spec:
      containers:
      - name: product-api
        image: registry.cn-hangzhou.aliyuncs.com/myorg/product-api:v2.1.0
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      - name: istio-proxy
        image: docker.io/istio/proxyv2:1.21.2
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

## 四、流量治理策略

### 4.1 Gateway 入口定义

```yaml
# gateway-product.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: product-gateway
  namespace: microservice-prod
spec:
  selector:
    istio: ingressgateway  # 指向 ingress gateway pod
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: product-tls-cert  # cert-manager 管理的证书
    hosts:
    - "product.example.com"
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "product.example.com"
    # HTTP 自动跳转 HTTPS
    tls:
      httpsRedirect: true
```

### 4.2 VirtualService 路由规则

```yaml
# virtualservice-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-virtualservice
  namespace: microservice-prod
spec:
  hosts:
  - product.example.com
  gateways:
  - product-gateway
  http:
  # 金丝雀发布: 90% v1, 10% v2
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
```

### 4.3 DestinationRule 服务版本定义

```yaml
# destinationrule-product.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-destination
  namespace: microservice-prod
spec:
  host: product-service.microservice-prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        outlierDetection:   # 被动故障检测
          consecutiveErrors: 5
          interval: 30s
          baseEjectionTime: 30s
          maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### 4.4 熔断与限流

```yaml
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
  namespace: microservice-prod
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
      http:
        http1MaxPendingRequests: 20
        http2MaxRequests: 50
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 60s
      maxEjectionPercent: 100
---
# 速率限制
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit-payment
  namespace: microservice-prod
spec:
  workloadSelector:
    labels:
      app: payment-service
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.local_ratelimit
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          value:
            stat_prefix: http_local_rate_limiter
            token_bucket:
              max_tokens: 100
              tokens_per_fill: 100
              fill_interval: 60s
```

### 4.5 mTLS 零信任安全

```yaml
# peerauthentication-mtls.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: microservice-prod
spec:
  mtls:
    mode: STRICT  # STRICT=强制mTLS, PERMISSIVE=兼容明文, DISABLE=关闭
---
# 细粒度 mTLS 策略（仅特定端口要求 mTLS）
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: payment-mtls
  namespace: microservice-prod
spec:
  selector:
    matchLabels:
      app: payment-service
  portLevelMtls:
    8080:    # 业务端口
      mode: STRICT
    15020:   # 管理平面
      mode: STRICT
    15090:   # Prometheus 指标
      mode: PERMISSIVE
```

```yaml
# authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-allow-orders
  namespace: microservice-prod
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/microservice-prod/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/payment/*"]
---
# 默认拒绝所有入站（白名单模式）
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: default-deny-all
  namespace: microservice-prod
spec: {}
```

## 五、跨集群 Service Mesh

### 5.1 多集群拓扑

```
┌─────────────────────────────────────────────────────┐
│                  Shared Control Plane                │
│                   (ASM/Istiod)                       │
├────────────┬──────────────┬──────────────────────────┤
│ Cluster A  │  Cluster B   │     Cluster C            │
│  (杭州)     │   (上海)      │      (北京)              │
│  ┌──────┐  │  ┌──────┐    │  ┌────────────────────┐ │
│  │Product│  │ │Order │    │  │Payment + Inventory │ │
│  │Cart   │  │ │User  │    │  │Checkout            │ │
│  └──────┘  │  └──────┘    │  └────────────────────┘ │
│  EastWest GW│  EastWest GW  │  EastWest GW           │
└────────────┴──────────────┴──────────────────────────┘
```

```yaml
# multi-cluster-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: eastwest-gateway
  namespace: istio-system
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
    hosts:
    - "*.local"   # 所有 .local 域名透传到对应集群
```

```yaml
# serviceentry-cross-cluster.yaml
# 在 Cluster A 中添加对 Cluster B 服务的引用
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: order-service-cluster-b
  namespace: microservice-prod
spec:
  hosts:
  - order-service.prod-b.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - number: 8080
    name: http
    protocol: HTTP
  resolution: DNS
  endpoints:
  - address: order-service.prod-b.svc.cluster.local   # 通过内部 DNS 解析
    ports:
      http: 8080
    locality: prod-b   # 标记目标集群
```

## 六、运维管理

### 6.1 日常巡检清单

```bash
# 1. 检查 Istio 组件健康
kubectl get pods -n istio-system

# 2. 检查 sidecar 注入率
istioctl analyze -n microservice-prod

# 3. 查看连接状态
istioctl proxy-status | head -20

# 4. 检查证书有效期
kubectl get secret -n istio-system | grep ca-signer
openssl x509 -in /etc/certs/cert-chain.pem -noout -enddate

# 5. 查看 Envoy 统计
kubectl exec -n microservice-prod \
  $(kubectl get pod -l app=product -o jsonpath='{.items[0].metadata.name}') \
  -c istio-proxy -- pilot-agent request GET stats/prometheus \
  | grep envoy_cluster_health | grep unhealthy

# 6. 网格诊断
istioctl x precheck
istioctl verify-install --revision prod
```

### 6.2 故障排查指南

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Sidecar 未注入 | 命名空间缺 label | `kubectl label ns <ns> istio-injection=enabled` |
| 双向 TLS 失败 | PeerAuth 不匹配 | 检查 STRICT/PERMISSIVE 模式一致性 |
| 路由不生效 | VirtualService priority | 按顺序匹配，第一个命中即停止 |
| OOM Killed (Envoy) | 连接池太大 | 调整 connectionPool 参数 |
| 证书过期 | Citadel/CACSR 超时 | `istioctl x sync secret --all` |

### 6.3 性能调优建议

```yaml
# 针对高吞吐场景优化 Envoy 配置
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: performance-tuning
  namespace: microservice-prod
spec:
  workloadSelector:
    labels:
      app: product-service
  configPatches:
  # 增加 HTTP/2 并发流
  - applyTo: NETWORK_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          common_http_protocol_options:
            headers_with_underscore_handling: ACCEPT
          http2_protocol_options:
            initial_stream_window_size: 65536    # 默认 16KB
            initial_connection_window_size: 1048576  # 默认 1MB
```

## 七、成本估算

| 资源 | 规格 | 月费用估算 |
|------|------|-----------|
| ASM Pro 版 (阿里云) | 企业版 | ¥1,800/月 |
| Istio Control Plane (K8s) | 2x4C8G | 自付 K8s 节点资源 |
| Envoy Sidecar | 每 Pod ~100m CPU/128MB | 无额外费用 |
| egress gateway | 2xC4G | ¥500/月 |

## 八、面试考点

1. **Sidecar 注入原理**: MutatingWebhookConfiguration 拦截 CREATE 请求，inject 扩展器追加 sidecar 容器
2. **XDS 协议**: CDS(Cluster)、EDS(Endpoints)、RDS(Route)、LDS(Listener) 四种发现服务
3. **mTLS 流程**: SPIFFE ID → SDS(secret discovery service) → 动态证书轮换（999小时有效期）
4. **流量切片**: VirtualService + DestinationRule subsets 配合 header/user-based routing
5. **故障隔离**: outlierDetection 的 consecutiveErrors + baseEjectionTime 机制

## 九、课后练习

1. 在本地 Minikube 部署 Istio，创建 3 个微服务并实现灰度发布
2. 编写 DestinationRule 实现基于响应时间的熔断策略
3. 配置 mTLS STRICT 模式，验证非 Sidecar Pod 被拒绝
4. 使用 kiali UI 分析调用链拓扑图
