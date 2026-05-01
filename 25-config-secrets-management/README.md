# 项目二十五: 云原生配置中心与密钥管理 (Nacos + Vault + KMS)

## 一、项目背景

应用配置和密钥管理的核心问题：
- **配置分散**: application.yml、环境变量、ConfigMap、Secret 散布各处，无法统一管理
- **热更新困难**: 修改配置需要重启 Pod → 服务中断、流量抖动
- **敏感信息泄露**: API Key/DB Password 直接硬编码在代码或环境变量中
- **环境隔离混乱**: dev/staging/prod 配置靠 copy-paste，容易出错
- **密钥轮换滞后**: TLS 证书/API Token 过期导致大面积故障

## 二、架构设计

### 2.1 配置与密钥管理全景图

```
┌──────────────────────────────────────────────────────────────┐
│                     Application Layer                         │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐             │
│  │ App 1  │  │ App 2  │  │ App 3  │  │ App N  │             │
│  │ OTel   │  │ OTel   │  │ OTel   │  │ OTel   │             │
│  │ Config  │  │ Config  │  │ Config  │  │ Config  │           │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘             │
│      │           │           │           │                    │
├──────┼───────────┼───────────┼───────────┼────────────────────┤
│      ▼           ▼           ▼           ▼                    │
│  ┌──────────────────────────────────────────────────────┐     │
│  │           Configuration & Secrets Bridge              │     │
│  │                                                       │     │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │     │
│  │  │ Config   │  │ Secret   │  │ Runtime            │ │     │
│  │  │ Center   │  │ Manager  │  │ Injector           │ │     │
│  │  │ Nacos    │  │ Vault    │  │                    │ │     │
│  │  │ Apollo   │  │ Sealed   │  │ Auto-Rotate        │ │     │
│  │  │ Consul   │  │ Secrets  │  │ Env Injection      │ │     │
│  │  └────┬─────┘  └────┬─────┘  └────────┬───────────┘ │     │
│  └───────┼──────────────┼────────────────┼─────────────┘     │
│          │              │                │                    │
│  ┌───────▼──────┐  ┌────▼────────┐  ┌───▼────────────┐      │
│  │ etcd/ZK     │  │ K8s etcd   │  │ Cloud KMS       │      │
│  │ (Config DB) │  │ (Secret St.)│  │ (Vault Backend) │      │
│  └─────────────┘  └─────────────┘  └────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 多云方案对比

| 维度 | Nacos | HashiCorp Vault | 阿里云 KMS | AWS Secrets Manager | 华为云 CSM |
|------|-------|----------------|-----------|--------------------|----------|
| 主要用途 | 配置中心 | 密钥管理 | KMS/HSM | 密钥存储 | 密钥管理 |
| 热更新 | ✅ N/A 推送 | ❌ 需轮询/注入 | ❌ | ❌ | ❌ |
| KV Store | ✅ 支持 | ⚠️ KV Engine | ❌ | ✅ Secrets | ❌ |
| 动态凭证 | ❌ | ✅ Database/RDS | ❌ | ❌ | ❌ |
| 加密即服务 | ❌ | ✅ Transit Engine | ✅ AES-256 | ✅ AES-256-GCM | ✅ SM4/AES |
| 审计日志 | ⚠️ 社区版 | ✅ Enterprise | ✅ ActionTrail | ✅ CloudTrail | ✅ HSS |
| 多租户 | ✅ Namespace | ✅ Namespaces | ✅ Key Aliases | ✅ Key Policies | ✅ Domain |

## 三、核心部署方案

### 3.1 Nacos 配置中心集群

```yaml
# nacos-cluster-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-cm
  namespace: config-center
data:
  # ==================== nacos 应用配置 ====================
  application.properties: |
    # Server
    server.servlet.contextPath=/nacos
    server.contextPath=/nacos
    server.port=8848
    
    # Database - MySQL 持久化配置
    spring.datasource.platform=mysql
    db.num=1
    db.url.0=jdbc:mysql://mysql-nacos.config-center.svc:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false
    db.user.0=${MYSQL_USER:nacos}
    db.password.0=${MYSQL_PASS:nacos}
    
    # Nacos 认证
    nacos.core.auth.enabled=true
    nacos.core.auth.server.identity.key=nacosServerKey
    nacos.core.auth.server.identity.value=nacosServerToken
    nacos.core.auth.plugin.nacos.token.secret.key=SecrectKey012345678901234567890123456789012345678901234567890123456789
    
    # Cluster 模式
    nacos.naming.empty-service.auto-clean=true
    nacos.naming.service.cleanup.interval=60000
    
    # Performance tuning
    nacos.core.protocol.dust.max-thread-count=10
    max.connections.per.host=1000
    
  # ==================== cluster.conf (集群节点列表) ====================
  cluster.conf: |
    nacos-0.nacos-headless.config-center.svc:8848
    nacos-1.nacos-headless.config-center.svc:8848
    nacos-2.nacos-headless.config-center.svc:8848
  
  # ==================== init.d/custom.properties ====================
  custom.properties: |
    management.metrics.export.elastic.enabled=false
    management.metrics.export.influx.enabled=false
---
# StatefulSet for Nacos cluster
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: config-center
spec:
  serviceName: nacos-headless
  replicas: 3
  selector:
    matchLabels:
      app: nacos
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9555"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nacos
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nacos
        image: nacos/nacos-server:v2.3.1
        ports:
        - containerPort: 8848  # HTTP
        - containerPort: 9848  # Client-RPC
        - containerPort: 9849  # Server-RPC
        env:
        - name: NACOS_REPLICAS
          value: "3"
        - name: MODE
          value: "cluster"
        - name: MYSQL_SERVICE_HOST
          valueFrom:
            configMapKeyRef:
              name: mysql-nacos-config
              key: host
        - name: MYSQL_SERVICE_DB_NAME
          value: "nacos_config"
        - name: MYSQL_SERVICE_USER
          valueFrom:
            secretKeyRef:
              name: nacos-db-creds
              key: username
        - name: MYSQL_SERVICE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: nacos-db-creds
              key: password
        - name: JVM_XMS
          value: "512m"
        - name: JVM_XMX
          value: "1g"
        - name: JVM_XMN
          value: "512m"
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        readinessProbe:
          httpGet:
            path: /nacos/actuator/health/readiness
            port: 8848
          initialDelaySeconds: 30
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /nacos/actuator/health/liveness
            port: 8848
          initialDelaySeconds: 60
          periodSeconds: 10
        volumeMounts:
        - name: config-volume
          mountPath: /home/nacos/conf/application.properties
          subPath: application.properties
        - name: config-volume
          mountPath: /home/nacos/conf/cluster.conf
          subPath: cluster.conf
      volumes:
      - name: config-volume
        configMap:
          name: nacos-cm
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: alicloud-disk-essd-pl0
      resources:
        requests:
          storage: 20Gi
---
# Headless Service (Pod DNS 发现)
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: config-center
spec:
  clusterIP: None
  ports:
  - port: 8848
    targetPort: 8848
    name: client
  - port: 9848
    targetPort: 9848
    name: client-rpc
  - port: 9849
    targetPort: 9849
    name: server-rpc
  selector:
    app: nacos
---
# Frontend Service (负载均衡访问)
apiVersion: v1
kind: Service
metadata:
  name: nacos
  namespace: config-center
spec:
  type: LoadBalancer
  ports:
  - port: 8848
    targetPort: 8848
  selector:
    app: nacos
```

### 3.2 Nacos Data Model - Group & Tenant & DataID

```
Config 三层隔离模型:
┌────────────────────────────────────────────────┐
│ Namespace (环境级: dev / staging / prod)         │
│  ┌──────────────────────────────────────────┐  │
│  │ Group (业务分组: DEFAULT_GROUP / infra)  │  │
│  │  ┌────────────────────────────────────┐  │
│  │  │ Data ID: product-service.yaml      │  │
│  │  │    = <prefix>-<profile>.<suffix>   │  │
│  │  │ prefix:  ${spring.application.name} │  │
│  │  │ profile: ${spring.profiles.active}  │  │
│  │  │ suffix:  yaml/json/properties/xml   │  │
│  │  └────────────────────────────────────┘  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘

命名空间示例:
dev:   开发测试环境（共享开发资源）
staging: 预发环境（镜像生产配置）
prod:  生产环境（独立权限管控）
public: 默认命名空间
```

### 3.3 Spring Boot 接入 Nacos

```yaml
# bootstrap.yaml - Spring Boot 读取 Nacos 配置
spring:
  application:
    name: product-api
  profiles:
    active: prod
  cloud:
    nacos:
      # 配置中心
      config:
        server-addr: nacos.config-center.svc:8848
        namespace: ${ENV:NAMESPACE:prod}  # 动态切换命名空间
        file-extension: yaml
        refresh-enabled: true
        enabled: true
        # 共享配置(跨服务共用)
        shared-configs:
        - data-id: common-base.yaml
          group: DEFAULT_GROUP
          refresh: true
        - data-id: db-common.yaml
          group: INFRA
          refresh: true
        # 扩展配置
        extension-configs:
        - data-id: product-cache.yaml
          group: PRODUCT
          refresh: true
      
      # 注册中心(如需服务发现)
      discovery:
        server-addr: nacos.config-center.svc:8848
        namespace: ${ENV:NAMESPACE:prod}
```

```java
// Java: @NacosValue 实现运行时热更新
@RestController
@RequestMapping("/config")
public class ConfigController {
    
    // @NacosValue(value = "...", refresh = true) 实现动态刷新
    @NacosValue(value = "${feature.darkMode:false}", refresh = true)
    private Boolean darkMode;
    
    @NacosValue(value = "${rate.limit.qps:100}", refresh = true)
    private Integer rateLimitQps;
    
    @NacosValue(value = "${cache.ttl.seconds:300}", refresh = true)
    private Integer cacheTTL;
    
    @GetMapping("/features")
    public Map<String, Object> getFeatures() {
        return Map.of(
            "darkMode", darkMode,
            "rateLimit", rateLimitQps,
            "cacheTTL", cacheTTL
        );
    }
}

// Python Flask 接入示例
from flask import Flask, jsonify
import nacos  # pypi: nacos-sdk-python

client = nacos.NacosClient(
    server_addresses="nacos.config-center.svc:8848",
    namespace="prod"
)

def get_config(data_id, group="DEFAULT_GROUP"):
    """获取并监听配置变化"""
    config = client.get_config(data_id=data_id, group=group)
    
    def on_config_change(change):
        print(f"[Nacos] Config updated: {change.data}")
        reload_application_config(json.loads(change.data))
    
    client.add_config_watcher(
        data_id=data_id,
        group=group,
        callback=on_config_change
    )
    return json.loads(config)

app = Flask(__name__)

@app.route('/config')
def config():
    app_config = get_config("product-service-prod.yaml")
    return jsonify(app_config)
```

### 3.4 HashiCorp Vault 企业级密钥管理

```yaml
# vault-injected-agent.yaml
# Vault Agent Injector Sidecar - 自动注入 Secret 到 Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: app-prod
spec:
  template:
    metadata:
      annotations:
        # ===== Vault Agent Injector 注解 =====
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "payment-app-role"
        
        # DB credentials from Vault
        vault.hashicorp.com/agent-inject-secret-db-credentials: database-creds/payment-production
        vault.hashicorp.com/agent-inject-template-db-credentials: |
          {{- with secret "database-creds/payment-production" -}}
          [database]
          driver = postgresql
          host = $RDS_ENDPOINT
          port = 5432
          database = product_db
          username = {{ .Data.username }}
          password = {{ .Data.password }}
          max_connections = 20
          {{- end }}
        
        # TLS cert from Vault PKI
        vault.hashicorp.com/agent-inject-cert-tls: api-gateway-tls/payment.example.com
        vault.hashicorp.com/agent-inject-template-cert-tls: |
          {{- with secret "pki-int/sign/payment.example.com" -}}
          -----BEGIN CERTIFICATE-----
          {{ .Data.certificate }}
          -----END CERTIFICATE-----
          {{- end }}
          -----BEGIN PRIVATE KEY-----
          {{ .Data.private_key }}
          -----END PRIVATE KEY-----
        
        # Root CA bundle
        vault.hashicorp.com/ca-cert-file: "/vault/secrets/ca.crt"
        
        # Agent 启动前的准备命令
        vault.hashicorp.com/agent-pre-populate-only: "true"
        vault.hashicorp.com/agent-init-image: "hashicorp/vault-k8s:0.18.0"
    spec:
      serviceAccountName: vault-agent-sa
      containers:
      - name: payment-api
        image: registry.example.com/payment-api:v2.0.0
        env:
        # 关键: 密码不通过 ENV 传递!
        # - name: DB_PASSWORD  ← 禁止!
        # 改为从文件读取 (由 Vault Agent 注入)
        command: ["/bin/sh", "-c"]
        args:
        - |
          source /vault/secrets/db-credentials
          exec java -jar /app/payment-api.jar \
            --db.username=$VAULT_DB_USERNAME \
            --db.password=$(cat /vault/secrets/db-credentials | grep '^password=' | cut -d= -f2)
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        volumeMounts:
        - name: vault-secrets
          mountPath: /vault/secrets
          readOnly: true
      - name: vault-agent
        image: hashicorp/vault:1.15.0
        env:
        - name: VAULT_ADDR
          value: "https://vault.vault.svc:8200"
        - name: VAULT_NAMESPACE
          value: "prod"
        securityContext:
          runAsNonRoot: true
          runAsUser: 100
      volumes:
      - name: vault-secrets
        emptyDir: {}
```

```json
// Vault ACL Policy - 定义细粒度访问控制
{
  "path": {
    "database-creds/*": {
      "capabilities": ["read"],
      "allowed_response_headers": []
    },
    "database-creds/production": {
      "operations": {
        "read": {
          "display_names": ["Payment App Production Access"],
          "description": "Read-only DB credentials for payment production"
        }
      }
    },
    "secret/data/payment/*": {
      "capabilities": ["read", "list"]
    },
    "auth/token/create": {
      "policies": ["payment-app"],
      "ttl": "1h",
      "num_uses": 1,
      "renewable": false
    }
  }
}
```

### 3.5 Sealed Secrets (GitOps 友好的秘密管理)

```yaml
# sealed-secrets-controller.yaml
# Sealed Secrets: 将 Kubernetes Secret 加密后提交到 Git
# controller 在集群内解密为原生的 Secret
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: payment-app-secrets
  namespace: app-prod
  annotations:
    sealedsecrets.bitnami.com/namespace: "app-prod"
    sealedsecrets.bitnami.com/cluster-wide: "false"
spec:
  encryptedData:
    db-password: AgBxZ...long-ciphertext-encoded-in-AESCBC-with-key-aes256...==
    api-key-stripe: AgDkF...another-long-ciphertext...==
    jwt-signing-key: AgCmP...yet-another-long-ciphertext...==
  template:
    metadata:
      name: payment-app-secrets
      namespace: app-prod
    data:
      db-password: ""       # Will be overwritten by controller
      api-key-stripe: ""
      jwt-signing-key: ""
```

### 3.6 密钥自动轮换策略

```bash
#!/bin/bash
# auto-rotate-credentials.sh - 密钥自动轮换脚本
set -euo pipefail

# 使用 Vault dynamic secrets 自动创建临时数据库凭证
rotate_db_credential() {
    local LEASE_ID
    LEASE_ID=$(vault write -field=lease_id \
        database-creds/production/static-root \
        rotation_period=3600)
    
    echo "New credential lease: $LEASE_ID"
    echo "Credential will expire in 1 hour"
    
    # 通知应用更新连接字符串
    notify_app_of_rotation "$LEASE_ID"
    
    # 设置续约提醒
    watch_lease_expiration "$LEASE_ID"
}

# Vault TTL 续约机制
watch_lease_expiration() {
    local lease_id=$1
    local ttl
    ttl=$(vault read -field=lease_ttl "sys/leases/lookup/$lease_id")
    local remaining=$(echo "$ttl" | awk '{print int($1/60)}')
    local half_life=$((remaining / 2))
    
    if [ $half_life -lt 1 ]; then
        renew_lease "$lease_id"
    else
        sleep "${half_life}s"
        watch_lease_expiration "$lease_id"
    fi
}

# 检测 Vault 是否可用，不可用时优雅降级
check_vault_availability() {
    local retry_count=0
    local max_retries=5
    
    while ! vault status > /dev/null 2>&1; do
        retry_count=$((retry_count + 1))
        if [ $retry_count -ge $max_retries ]; then
            log_error "Vault unavailable after $max_retries retries"
            exit 1
        fi
        echo "[WARNING] Vault unavailable, retrying in 5s... ($retry_count/$max_retries)"
        sleep 5
    done
}

check_vault_availability
rotate_db_credential
```

## 四、运维管理

### 4.1 日常巡检清单

```bash
# 1. Nacos 健康检查
curl -s http://nacos.config-center.svc:8848/nacos/v1/cs/health | jq '.'

# 2. Nacos 配置数量统计
curl -s "http://nacos.config-center.svc:8848/nacos/v1/config/all-config? pageNo=1&pageSize=1000" \
  | jq '.pageItems | length'

# 3. Vault 密封状态
vault status

# 4. Sealed Secrets Controller 日志
kubectl logs -l app.kubernetes.io/name=sealed-secrets-controller -n kube-system -f --tail=100

# 5. 未旋转的密钥告警
# 查询所有 Vault static credential 的最后使用时间
vault list database/roles/payment-production/rotation_statements
```

### 4.2 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Nacos 集群脑裂 | Network split / etcd 异常 | 检查 cluster.conf 格式; 确保奇数个节点 |
| Vault unseal 超时 | 分享密钥未完全收集 | 至少 3/5 or 3/3 share keys needed |
| Pod 卡 Wait (Vault) | Vault Server 不可达 | `kubectl logs vault-agent -c vault-agent` |
| SealedSecret reconcile fail | 密钥不匹配 (cluster 变了) | 重新安装 sealed-secrets-controller |
| 配置热更新未生效 | Nacos 客户端未加 refresh 注解 | 添加 `@RefreshScope` / `refresh: true` |

## 五、成本估算

| 组件 | 规格 | 月费用 |
|------|------|--------|
| Nacos (3节点) | 2xC4G + MySQL | ¥800/月 |
| Vault (HA) | 2xC4G + Raft | ¥600/月 |
| Sealed Secrets Controller | 免费开源 | 自付资源 |
| 阿里云 KMS | 按量计费 | ¥50/月 |
| Nacos 商业版 (可选) | 企业版 | ¥500/月 |

## 六、面试考点

1. **Config vs Secret**: Config 是非敏感配置(Nacos/Apollo); Secret 是敏感凭据(Vault/K8s Secrets)
2. **Nacos 配置热更新原理**: Nacos 长轮询(long polling, ~30s)检测配置变更 → Push → 客户端回调
3. **Vault Dynamic Credentials**: 按需生成一次性 DB 用户(TTL到期自动DROP)，避免长期有效凭证
4. **Sealed Secrets**: 非对称加密(公钥公开可加密, 私钥仅 controller 持有), 支持 GitOps
5. **Encryption at Rest**: Vault 用 Shamir Secret Sharing 分片 master key; KMS 用 KMIP/BCE 标准

## 七、课后练习

1. 搭建 Nacos 集群并通过 Spring Boot 实现配置热更新
2. 部署 Vault HA 集群并完成 Database Secrets Engine 动态凭证创建
3. 使用 Sealed Secrets 将 K8s Secret 安全提交到 Git 仓库
4. 编写 Helm Chart 的 secrets.tpl 模板集成 Vault Agent Injector
