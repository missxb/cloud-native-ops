# 项目二十二: 多云统一身份管理与单点登录 (OIDC/IAM Federation)

## 一、项目背景

企业 IT 基础设施的治理挑战：
- **账号孤岛**: RAM(IAM)、AD(LDAP)、云原生 IAM 各自独立管理
- **权限爆炸**: 每个员工在不同云平台有不同的角色和权限
- **合规审计困难**: 无法统一追踪"谁在什么时候做了什么操作"
- **开发效率低**: 每次切换环境需要重新配置凭证

解决方案：建立统一的身份管理平台，实现 SSO + RBAC + Audit 一体化。

## 二、架构设计

### 2.1 统一身份架构

```
┌───────────────────────────────────────────────────────────┐
│                     Corporate Directory                    │
│              (Azure AD / Okta / Keycloak)                  │
└──────┬────────────────────────────┬───────────────────────┘
       │ SAML 2.0 / OIDC            │ OAuth2.0 / OpenID
       ▼                            ▼
┌─────────────┐           ┌────────────────┐
│  Keycloak   │◄──────────┤ Identity       │
│  (Internal  │  Trust    │ Federation     │
│   IdP)      │  Anchor   │ Service        │
└──────┬──────┘           └────────┬───────┘
       │                           │
       │     Kubernetes             │     Cloud Providers
       │     RBAC                   │     (RAM/IAM/OSS/...)
       ▼                           ▼
┌─────────────────────────────────────────────────────────┐
│         Authentication & Authorization Bridge            │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │ K8s AuthN   │  │ Cloud AuthZ │  │ Audit Log    │   │
│  │ webhook     │  │ mapping     │  │ Aggregator   │   │
│  └─────────────┘  └─────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 多云身份服务对比

| 维度 | 阿里云 RAM | AWS IAM | 华为云 IAM | 腾讯云 CAM | Keycloak (自建) |
|------|-----------|---------|-----------|-----------|----------------|
| 多用户管理 | ✅ | ✅ | ✅ | ✅ | ✅ |
| SAML/SOIDC | ✅ | ✅ | ✅ | ✅ | ✅ (核心功能) |
| LDAP 对接 | ⚠️ 部分 | ✅ | ✅ | ✅ | ✅ (内置) |
| MFA | ✅ | ✅ | ✅ | ✅ | ✅ |
| 细粒度策略 | JSON Policy | JSON Policy | JSON Policy | JSON Policy | Role-based |
| Audit 日志 | ActionTrail | CloudTrail | CloudTrace | 云审计 | ⚠️ 需集成 |

## 三、核心部署方案

### 3.1 Keycloak 集群部署

```yaml
# keycloak-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-config
  namespace: identity
data:
  keycloak.conf: |
    # ==================== 基础配置 ====================
    http-relative-path=/auth
    http-port=8080
    
    # ==================== 数据库 ====================
    db-url-host=postgresql-keycloak.identity.svc
    db-url-port=5432
    db-url-db=keycloak
    db-username=${KC_DB_USER}
    db-password=${KC_DB_PASS}
    
    # ==================== HA 配置 ====================
    spi-experimental-cli-secret-hash-enabled=true
    hostname-strict=false
    hostname-strict-backchannel=false
    
    # ==================== Session 设置 ====================
    session-max-lifetime=3600
    offline-session-max-lifetime=31536000
    login-lifespan=3600
    access-token-lifespan=600  # 短命 token 安全加固
    
    # ==================== 资源限制 ====================
    jvm-memory-limit=2g
    
    # ==================== TLS ====================
    ssl-required=external
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
  namespace: identity
spec:
  serviceName: keycloak-headless
  replicas: 3
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - keycloak
            topologyKey: kubernetes.io/hostname  # 不同节点部署
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:23.0
        args: ["start", "--profile=default"]
        env:
        - name: KC_DB
          value: postgres
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: keycloak-db-credentials
              key: username
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-db-credentials
              key: password
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-admin-creds
              key: password
        - name: KC_HTTP_RELATIVE_PATH
          value: "/auth"
        resources:
          requests:
            cpu: "1000m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        readinessProbe:
          httpGet:
            path: /auth/health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /auth/health/ready
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
      volumes:
      - name: keycloak-config
        configMap:
          name: keycloak-config
---
# Headless Service (用于 Pod DNS 解析)
apiVersion: v1
kind: Service
metadata:
  name: keycloak-headless
  namespace: identity
spec:
  clusterIP: None
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: keycloak
---
# Ingress (HTTPS 入口)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak-ingress
  namespace: identity
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "4m"
spec:
  tls:
  - hosts:
    - auth.example.com
    secretName: auth-tls-cert
  rules:
  - host: auth.example.com
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: keycloak
            port:
              number: 8080
```

### 3.2 Realm & Client 配置

```json
// keycloak-realm.json - 企业 realm 配置示例
{
  "realm": "enterprise-production",
  "enabled": true,
  "sslRequired": "external",
  
  // Token 安全配置
  "accessCodeLifespan": 300,
  "sessionIdleTimeout": 3600,
  "accessTokenLifespan": 600,
  "ssoSessionIdleTimeout": 7200,
  
  // 密码策略
  "passwordPolicy": "length(12) and notUsername and lowerCase(1) and upperCase(1) and digits(1) and specialCharacters(1)",
  
  // MFA 强制启用
  "requiredActions": ["CONFIGURE_TOTP"],
  "otpPolicyType": "totp",
  "otpPolicyDigits": 6,
  "otpPolicyPeriod": 30,
  "otpPolicyAlgorithm": "HmacSHA1",
  
  // 事件监听器
  "eventsListeners": ["jboss-logging"],
  "enableEventLogging": true,
  
  "clients": [
    {
      "clientId": "kubernetes-dashboard",
      "protocol": "openid-connect",
      "publicClient": false,
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": true,
      "authorizationServicesEnabled": true,
      "rootUrl": "https://k8s-dashboard.example.com",
      "baseUrl": "/",
      "validRedirectUris": ["https://k8s-dashboard.example.com/*"],
      "attributes": {
        "id.token.as.detached.signature": "false",
        "saml.assertion.signature": "false",
        "client.secret.creation.time": "1699000000",
        "login.url.template": "",
        "claim.always.add.to.token.enabled": "true",
        "display.on.consent.screen": "false",
        "oauth2.device.authorization.grant.enabled": "false"
      },
      "defaultClientScopes": ["web-origins", "role_list"],
      "optionalClientScopes": ["address", "phone", "offline_access", "microprofile-jwt"]
    },
    {
      "clientId": "terraform-backend",
      "protocol": "openid-connect",
      "publicClient": false,
      "standardFlowEnabled": false,
      "serviceAccountsEnabled": true,
      "directAccessGrantsEnabled": true,
      "serviceRoles": ["service-account"],
      "attributes": {
        "token.response.type.bearer.lower-case": "true"
      }
    }
  ]
}
```

### 3.3 Kubernetes RBAC 与 Identity 映射

```yaml
# rbac-mapping.yaml - Keycloak Roles → K8s Groups 映射
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-admin-group
subjects:
- kind: Group
  name: enterprise:cloud-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-developer-namespaces
subjects:
- kind: Group
  name: enterprise:developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
---
# 自定义 ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-role
rules:
- apiGroups: ["apps", "extensions"]
  resources: ["deployments", "daemonsets", "statefulsets", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
  resourceNames: []
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch"]
```

### 3.4 Webhook AuthN Controller (K8s ↔ Keycloak)

```go
// webhook-authn-controller.go (简化版概念代码)
package main

import (
    "encoding/json"
    "net/http"
    
    "github.com/coreos/go-oidc/v3/oidc"
    "golang.org/x/oauth2"
    v1 "k8s.io/api/authentication/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type AuthReviewRequest struct {
    TypeMeta   metav1.TypeMeta   `json:"kind"`
    APIVersion string            `json:"apiVersion"`
    Spec       AuthReviewSpec    `json:"spec"`
}

type AuthReviewSpec struct {
    Extra map[string]json.RawMessage `json:"extra"`
    Scopes []string                   `json:"scopes"`
    Username string                    `json:"username,omitempty"`
}

var (
    issuerURL    = "https://auth.example.com/auth/realms/enterprise-production"
    provider     *oidc.Provider
    verifier     *oidc.IDTokenVerifier
    oidcConfig   oauth2.Config
)

func init() {
    var err error
    provider, err = oidc.NewProvider(context.Background(), issuerURL)
    if err != nil {
        panic(err)
    }
    verifier = provider.Verifier(&oidc.Config{ClientID: oidcConfig.ClientID})
}

// TokenReviewWebhook handles authentication token reviews from kube-apiserver
func TokenReviewWebhook(w http.ResponseWriter, r *http.Request) {
    var reviewReq AuthReviewRequest
    json.NewDecoder(r.Body).Decode(&reviewReq)
    
    // Extract Bearer token from request headers or spec
    authHeader := r.Header.Get("Authorization")
    token := strings.TrimPrefix(authHeader, "Bearer ")
    
    // Verify ID token with Keycloak
    idToken, err := verifier.Verify(context.Background(), token)
    if err != nil {
        sendDenied(w, "invalid token: "+err.Error())
        return
    }
    
    // Extract groups and roles from ID token claims
    claims := struct {
        Groups  []string `json:"groups"`
        Roles   []string `json:"roles"`
        Email   string   `json:"email"`
        Sub     string   `json:"sub"`
    }{}
    if err := idToken.Claims(&claims); err != nil {
        sendDenied(w, "cannot parse claims: "+err.Error())
        return
    }
    
    // Map OIDC groups to Kubernetes groups
    k8sGroups := mapOIDCGroupsToK8S(claims.Groups)
    
    // Build positive auth response
    response := v1.TokenReview{
        TypeMeta: metav1.TypeMeta{Kind: "TokenReview", APIVersion: "authentication.k8s.io/v1"},
        Status: v1.TokenReviewStatus{
            Authenticated: true,
            User: v1.UserInfo{
                Username: fmt.Sprintf("keycloak:%s", claims.Sub),
                Groups:   k8sGroups,
                Extra: map[string]v1.ExtraValue{
                    "groups":   v1.ExtraValue(k8sGroups),
                    "email":    v1.ExtraValue{claims.Email},
                    "realm":    v1.ExtraValue{"enterprise-production"},
                },
            },
        },
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func mapOIDCGroupsToK8S(oidcGroups []string) []string {
    groupMapping := map[string][]string{
        "cloud-admin":  {"system:masters", "enterprise:k8s-admin"},
        "cloud-developer": {"enterprise:k8s-developer"},
        "cloud-viewer":    {"enterprise:k8s-viewer"},
    }
    
    var result []string
    for _, g := range oidcGroups {
        if mapped, ok := groupMapping[g]; ok {
            result = append(result, mapped...)
        }
    }
    return result
}

func sendDenied(w http.ResponseWriter, reason string) {
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(v1.TokenReview{Status: v1.TokenReviewStatus{Authenticated: false, Reason: reason}})
}
```

## 五、跨云 IAM 联邦

```bash
#!/bin/bash
# create-cloud-federation.sh - 创建云厂商联邦信任关系
set -euo pipefail

KEYCLOCK_ISSUER="https://auth.example.com/auth/realms/enterprise-production"
KEYCLOCK_WELL_KNOWN="$KEYCLOCK_ISSUER/.well-known/openid-configuration"

echo "=== 创建云厂商 IAM 联邦 ==="

# 1. 获取 Keycloak 的 JWKS endpoint
JWKS_URL=$(curl -s $KEYCLOCK_WELL_KNOWN | jq -r '.jwks_uri')
echo "Keycloak JWKS: $JWKS_URL"

# 2. 创建阿里云 RAM OIDC 身份提供商
aliyun ram CreateOpenIdConnectProvider \
    --DisplayName "Keycloak-Enterprise" \
    --URL "$KEYCLOCK_ISSUER" \
    --ThumbprintList "" \
    --IssuerUrl "$KEYCLOCK_ISSUER" \
    --Audience "kubernetes-dashboard,terraform-backend" \
    --Comment "Federated identity provider for Kubernetes and Terraform"

# 3. 创建信任策略角色
aliyun ram CreateRole \
    --RoleName "k8s-deploy-role" \
    --AssumeRolePolicyDocument '{
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Effect": "Allow",
                "Principal": {
                    "Federated": ["acs:ram::123456789012:oidc-provider/keycloak-enterprise"]
                }
            }
        ],
        "Version": "1"
    }'

# 4. 绑定权限策略
aliyun ram AttachPolicyToRole \
    --PolicyType Custom \
    --PolicyName "ECSFullAccess,K8sFullAccess" \
    --RoleName "k8s-deploy-role"

echo "✅ IAM Federation created successfully!"
```

## 六、成本估算

| 组件 | 规格 | 月费用 |
|------|------|--------|
| Keycloak (3节点) | 2xC4G | ¥600/月 |
| PostgreSQL (Keycloak DB) | 2C4G | ¥200/月 |
| 阿里云 RAM | 免费 | ¥0 |
| 云 WAF (保护认证页面) | 基础版 | ¥300/月 |
| MFA (短信验证码) | 按量 | ¥100/月 |

## 七、面试考点

1. **OIDC vs SAML**: OIDC 基于 JWT 轻量级、适合 SPA; SAML 基于 XML 更重量级、适合企业级
2. **JWT 结构**: header(alg/kty) + payload(sub/iss/iat/exp) + signature(HMAC/RSA)
3. **RBAC 模型**: Role → RoleBinding → Subject(Users/Groups/ServiceAccounts)
4. **Trust Boundary**: Identity Provider 是信任锚点，所有下游系统向其验证 token 有效性
5. **Token 轮换**: Access Token 短命(5min) + Refresh Token 长效(7天) + JTI 防重放

## 八、课后练习

1. 搭建 Keycloak 集群并完成 SAML 协议对接 Azure AD
2. 将 Keycloak 用户组映射为 K8s Group 并配置 RBAC
3. 使用 terraform-aws-provider 配置 IAM OIDC Federation
4. 编写审计日志采集脚本收集 Keycloak + K8s + 云厂商日志到 Elasticsearch
