# 项目二十三: 云原生 API 网关与后端服务编排

## 一、项目背景

微服务架构中 API 治理的核心问题：
- **入口混乱**: 每个微服务独立暴露端口，外部调用无处下手
- **安全缺口**: 没有统一的鉴权/限流/WAF 防护
- **灰度困难**: 按用户/地域/版本路由需要复杂的 Nginx 配置
- **监控盲区**: API 的 QPS、延迟、错误率缺乏统一度量

API Gateway 是微服务的统一入口，承担路由、鉴权、限流、监控等横切关注点。

## 二、架构设计

### 2.1 API 网关全景图

```
┌──────────────────────────────────────────────────────────────┐
│                       External Users                        │
│   Web App  │  Mobile App  │  Partner API  │  IoT Devices    │
└──────┬─────────────────────────────────────┬────────────────┘
       │                                     │
       ▼                                     ▼
┌──────────────────────────────────────────────────────────────┐
│              Cloud API Gateway (WAF + DDoS)                  │
│           ALB/NLB/WAF / AWS API Gateway / GCP IAP            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│             Ingress Controller Layer                         │
│         Nginx Ingress / HAProxy / Kong / APISIX             │
│                                                              │
│  ┌─────────────┐  ┌────────────┐  ┌──────────────────┐     │
│  │ Rate Limit   │  │ AuthN/Z    │  │ TLS Termination  │     │
│  │ & Quota      │  │ JWT/OAuth  │  │ Certificate Mgmt │     │
│  └─────────────┘  └────────────┘  └──────────────────┘     │
└────────────────────────┬────────────────────────────────────┘
                         │ Kubernetes Ingress
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Backend Services                           │
│  Product-Svc │ Order-Svc │ Payment-Svc │ User-Svc │ Search   │
│  v1,v2,v3    │ v1,v2     │ v1          │ v1       │ v1,v2    │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 网关技术选型对比

| 特性 | APISIX | Kong | Nginx Ingress | Traefik | Envoy |
|------|--------|------|--------------|---------|-------|
| 语言 | OpenResty(Lua) | Node.js/C++ | C | Go | C++ |
| 性能 | ★★★★★ 高并发 | ★★★ 中等 | ★★★★★ | ★★★★ 高 | ★★★★★ 极高 |
| 动态配置 | ✅ etcd | ✅ DB | ⚠️ reload | ✅ CRD | ✅ xDS |
| 插件生态 | 100+ | 30+ | N/A | N/A | FilterChain |
| 多租户 | ✅ 原生 | ⚠️ 需付费 | ❌ | ❌ | ⚠️ 复杂 |
| 可观测性 | Prometheus | Prometheus | Basic | Basic | Stats/Dump |
| GraphQL | ✅ 支持 | 社区插件 | ❌ | ❌ | 需定制 |

## 三、核心部署方案

### 3.1 Apache APISIX 集群部署

```yaml
# apisix-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: apisix-config
  namespace: api-gateway
data:
  config.yaml: |
    deployment:
      admin:
        allow_admin:
          - 127.0.0.0/24
          - ::1/128
        keys:
          editor: ${APISIX_EDITOR_KEY}
          viewer: ${APISIX_VIEWER_KEY}
      
      etcd:
        host:
          - "http://etcd-0.etcd.api-gateway.svc:2379"
          - "http://etcd-1.etcd.api-gateway.svc:2379"
          - "http://etcd-2.etcd.api-gateway.svc:2379"
        prefix: "/apisix"
        timeout: 30
      
      proxy_cache: on  # 启用响应缓存
      
      dashboard:
        enable: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apisix
  namespace: api-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apisix
  template:
    metadata:
      labels:
        app: apisix
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
      containers:
      - name: apisix
        image: apache/apisix:3.9.0-debian
        env:
        - name: APISIX_ENGINE_KEEPALIVE_IDLE_TIMEOUT
          value: "60"
        - name: APISIX_ENGINE_MAX_MEMORY_POOL_SIZE
          value: "104857600"  # 100MB
        ports:
        - containerPort: 9080  # HTTP API
        - containerPort: 9443  # HTTPS
        - containerPort: 9091  # Dashboard
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /apisix/admin/test-token
            port: 9080
            httpHeaders:
            - name: X-Token
              value: "${APISIX_EDITOR_KEY}"
          initialDelaySeconds: 10
          periodSeconds: 5
---
# Service (负载均衡)
apiVersion: v1
kind: Service
metadata:
  name: apisix-admin
  namespace: api-gateway
spec:
  type: ClusterIP
  ports:
  - port: 9080
    targetPort: 9080
  - port: 9091
    targetPort: 9091
  selector:
    app: apisix
---
# NodePort for external access (测试环境用)
apiVersion: v1
kind: Service
metadata:
  name: apisix-gateway
  namespace: api-gateway
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 9080
    name: http
  - port: 443
    targetPort: 9443
    name: https
  selector:
    app: apisix
```

### 3.2 Route 与插件配置

```bash
#!/bin/bash
# configure-apisix-routes.sh - APISIX 路由配置脚本
set -euo pipe fail

APISIX_ADMIN_URL="http://apisix-admin:9080/apisix/admin"
ADMIN_TOKEN="${APISIX_EDITOR_KEY:?Token required}"

# ==================== 1. 定义消费者(auth-key 插件) ====================
echo "Creating consumers..."
curl -s "$APISIX_ADMIN_URL/consumers" \
  -H "X-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "partner-api",
    "auth_key": "partner-api-key-2024",
    "plugins": {
      "key-auth": {
        "key": "partner-api-key-2024"
      }
    }
  }'

curl -s "$APISIX_ADMIN_URL/consumers" \
  -H "X-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "web-app",
    "auth_jwt": {
      "key": "my-jwt-signing-key",
      "algorithm": "HS256",
      "exp": 86400
    }
  }'

# ==================== 2. 创建上游(upstream)====================
echo "Creating upstreams..."
curl -s "$APISIX_ADMIN_URL/upstreams/product-upstream" \
  -H "X-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "desc": "Product service upstream",
    "type": "roundrobin",
    "nodes": {
      "product-service.app-prod.svc:8080": 1
    },
    "retries": 3,
    "timeout": {
      "connect": 5000,
      "send": 10000,
      "read": 30000
    },
    "health_check": {
      "active": {
        "timeout": 5,
        "concurrency": 10,
        "http_path": "/health",
        "healthy": {
          "interval": 20,
          "successes": 3
        },
        "unhealthy": {
          "interval": 10,
          "http_statuses": [429, 404, 500, 501, 502, 503, 504, 500]
        }
      }
    }
  }'

# ==================== 3. 创建带金丝雀发布的 Route ====================
echo "Creating canary route..."
curl -s "$APISIX_ADMIN_URL/routes/canary-product-route" \
  -H "X-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "/api/products/*",
    "upstream_id": "product-upstream",
    "methods": ["GET", "POST"],
    "priority": 10,
    "vars": [
      ["arg_version", "==", "v2"]
    ],
    "plugins": {
      "limit-count": {
        "count": 1000,
        "time_window": 60,
        "rejected_code": 429,
        "key": "remote_addr"
      },
      "response-rewrite": {
        "body": "{\"message\": \"deprecated\", \"version\": \"v1\"}",
        "headers": {
          "X-RateLimit-Limit": "1000",
          "X-RateLimit-Remaining": "999"
        },
        "status_code": 200,
        "variables": [["get_args_version", "=", "v1"]]
      }
    },
    "upstream": {
      "type": "roundrobin",
      "nodes": {
        "product-service-v1.app-prod.svc:8080": 90,
        "product-service-v2.app-prod.svc:8080": 10
      }
    }
  }'

# ==================== 4. 灰度发布基于 Header 的用户分片 ====================
echo "Creating user-based routing..."
curl -s "$APISIX_ADMIN_URL/routes/user-segmented-route" \
  -H "X-Token: $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "/api/orders/*",
    "upstream_id": "order-upstream",
    "plugins": {
      "proxy-mirror": {
        "mirror_uri": "/debug/order-cache",
        "mirror_body": true
      },
      "real-ip": {
        "source": ["X-Forwarded-For"],
        "recursive_search": true,
        "nested_address": false
      }
    },
    "strip_query_string": true
  }'

echo "✅ Routes configured successfully!"
```

### 3.3 Nginx Ingress + APISIX 双层架构

```yaml
# ingress-apisix-layered.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: api-gateway
  annotations:
    # Nginx Ingress 负责 L7 基础功能
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    # 转发到 APISIX 做高级流量治理
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: apisix-gateway
            port:
              number: 80

# ==================== APISIX 高级路由 ====================
# 在 APISIX 中配置的 Route 对象(通过 Admin API 或 CRD):

# apisixroute-crd.yaml
apiVersion: apisix.apache.org/v2beta1
kind: ApisixRoute
metadata:
  name: product-canary-route
  namespace: app-prod
spec:
  http:
  - name: product-v1-default
    match:
      paths:
      - /api/products/*
      backends:
      - serviceName: product-service-v1
        servicePort: 8080
    plugins:
      limit-count:
        count: 500
        timeWindow: 60
        rejectedCode: 429
  - name: product-v2-canary
    match:
      headers:
        x-canary-version:
          exact: "enabled"
      backends:
      - serviceName: product-service-v2
        servicePort: 8080
      plugins:
        prometheus: {}
        response-rewrite:
          headers:
            x-routing-version: "v2-canary"
```

### 3.4 自定义 Lua 插件

```lua
-- plugins/auth-signature/auth.lua
-- API 签名认证插件（防篡改）
local hmac = require "resty.hmac"
local sha256 = require "resty.sha256"
local base64 = require "resty.base64"
local core = require "apisix.core"
local string = string
local tonumber = tonumber

local _M = {}
_M.version = 1.0
_M.priority = 1  -- 高优先级先执行

function _M.filter(conf, ctx)
    local method = core.request.method()
    local uri = core.request.uri()
    
    -- 获取签名相关 header
    local timestamp = core.request.header("X-Timestamp")
    local nonce = core.request.header("X-Nonce") or ""
    local signature = core.request.header("X-Signature") or ""
    local app_id = core.request.header("X-App-Id") or ""
    
    if not timestamp or not app_id then
        return ngx.exit(401), {error_msg = "Missing authentication headers"}
    end
    
    -- 验证时间戳有效期 (±5分钟)
    local now = ngx.time()
    if math.abs(now - tonumber(timestamp)) > 300 then
        return ngx.exit(401), {error_msg = "Timestamp expired"}
    end
    
    -- 防止重放攻击 (检查 nonce 是否在 Redis 中已存在)
    -- (此处省略 Redis 检查逻辑)
    
    -- 构造签名字符串: APP_ID + METHOD + URI + TIMESTAMP + NONCE
    local sign_data = app_id .. method .. uri .. tostring(timestamp) .. nonce
    
    -- 计算 HMAC-SHA256
    local secret = conf.secret_keys[app_id] or conf.default_secret
    local expected_sig = hmac.new(sha256.digest, secret)
    expected_sig:update(sign_data)
    expected_sig = base64.encode(expected_sig:digest(), true, true)
    
    if signature ~= expected_sig then
        return ngx.exit(403), {error_msg = "Invalid signature"}
    end
    
    return nil  -- 认证通过
end

return _M
```

```bash
# 注册自定义插件
curl -s "http://apisix-admin:9080/apisix/admin/plugin_configs/signature-auth" \
  -H "X-Token: $EDITOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label_expr": "true",
    "rule_expr": "true",
    "secret_keys": {
      "partner-api": "shared-secret-2024",
      "internal-svc": "internal-secret"
    },
    "default_secret": "fallback-secret",
    "nonces_ttl": 300
  }'
```

## 四、运维管理

### 4.1 日常巡检清单

```bash
# 1. APISIX 健康检查
curl -s http://localhost:9080/apisix/admin/workers --header "X-Token: $TOKEN" | jq '.[] | {id, uptime}'

# 2. 各路由 QPS (Prometheus metrics)
curl -s http://localhost:9091/metrics | grep apisix_http_requests_total

# 3. 连接数统计
curl -s http://localhost:9187/status | grep active_connections

# 4. 慢查询日志
tail -f /usr/local/apisix/logs/delay.log
```

### 4.2 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 502 Bad Gateway | 后端服务不可达 | `curl http://backend:port/health` |
| 504 Gateway Timeout | 后端处理超时 | 调整 proxy-read/send timeout |
| 限流频繁触发 | limit-count 阈值过低 | 根据实际 QPS 调整 count 参数 |
| 证书过期 | cert-manager 未续期 | `kubectl get certificate -A` |

## 五、成本估算

| 组件 | 规格 | 月费用 |
|------|------|--------|
| APISIX (3节点) | 2xC4G | ¥600/月 |
| Nginx Ingress | 免费开源 | 自付 K8s 资源 |
| 阿里云 API Gateway | 企业版 | ¥2,000/月 |
| SSL 证书 | OV DV 多年 | ¥200/年 |

## 六、面试考点

1. **API Gateway 职责**: 统一入口、鉴权、限流、协议转换、监控、灰度发布
2. **Canary vs Blue/Green**: Canary 逐步放量(按权重/header); Blue-Green 全量切换两个环境
3. **JWT 验签流程**: 解析 header → 查找 public key/JWKS → 验证 signature → 检查 exp/aud/nbf
4. **Rate Limiting 算法**: Token Bucket(漏桶)、Sliding Window、Fixed Window 各有优缺点
5. **Webhook + Nginx**: Ingress controller 将 YAML/Annotation 转为 Nginx.conf 并 reload

## 七、课后练习

1. 部署 APISIX 集群并完成 JWT 鉴权和限流插件配置
2. 实现基于 header 的用户分片金丝雀发布
3. 编写自定义 Lua 插件实现 API 签名认证
4. 使用 Grafana + Prometheus 搭建 API Gateway 仪表盘
