# 项目09：基于 ArgoCD 多集群 GitOps

> **云平台**: 华为云 CCE + 阿里云 ACK | **难度**: ⭐⭐⭐⭐ | **工具**: ArgoCD, Helm, Kustomize, Argo Rollouts  
> **场景**: 在华为云和阿里云管理多套 K8s 集群，通过 GitOps 实现一推全发

---

## 📖 项目概述

某企业 SaaS 在华为云 (国内) 和阿里云 (海外) 各部署多套 K8s 集群。通过 ArgoCD ApplicationSet + 统一 Helm Chart 实现跨云 GitOps。

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ArgoCD Multi-Cluster GitOps                      │
│                                                                      │
│  ┌──────────────┐   ┌──────────────┐                                │
│  │  Git Repo    │   │ OCI Registry │  ← Helm Chart + 镜像            │
│  │ (k8s-config) │   │ (Huawei SWR/ │                                 │
│  └──────┬───────┘   │  Aliyun ACR) │                                 │
│         │           └──────┬───────┘                                │
│         └──────────┬───────┘                                        │
│                    ▼                                                 │
│           ┌────────────────┐                                        │
│           │   ArgoCD Hub    │  ← 管理中心 (ACK)                     │
│           │ (ApplicationSet)│                                       │
│           └───┬──────┬──────┘                                       │
│               │      │                                               │
│   ┌───────────┴┐    ┌┴───────────┐    ┌───────────┐                │
│   │ 华为云 CCE  │    │ 阿里云 ACK  │    │ 阿里云 ACK │               │
│   │ (prod-cn)  │    │ (prod-sea) │    │ (staging) │               │
│   ├────────────┤    ├────────────┤    ├───────────┤               │
│   │ ArgoCD     │    │ ArgoCD     │    │ ArgoCD    │               │
│   │ Rollouts   │    │ Rollouts   │    │ Rollouts  │               │
│   │ CCE Addons │    │ ACK Addons │    │ ACK Addons│               │
│   └────────────┘    └────────────┘    └───────────┘               │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │   Argo Rollouts: 金丝雀/蓝绿发布 + 自动指标分析            │      │
│  └───────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: Hub ArgoCD 安装 (阿里云 ACK)

```bash
# ArgoCD 安装
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 配置外部 Secret Store
kubectl create secret generic argocd-repo-creds \
  -n argocd \
  --from-literal=url=https://github.com/company/k8s-config.git \
  --from-literal=username=gitops-bot \
  --from-literal=password=${GITHUB_TOKEN}

# 注册远程集群 (华为云 CCE)
argocd cluster add cce-prod-cn \
  --kubeconfig=/path/to/huawei-cce-kubeconfig \
  --name=hcc-cn-prod \
  --label=cloud=huawei \
  --label=env=prod \
  --label=region=cn-north-4

# 注册阿里云 ACK
argocd cluster add ack-prod-sea \
  --kubeconfig=/path/to/aliyun-ack-kubeconfig \
  --name=ack-sea-prod \
  --label=cloud=aliyun \
  --label=env=prod \
  --label=region=ap-southeast-1
```

### Step 2: ApplicationSet 统一管理

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: saas-platform
  namespace: argocd
spec:
  # 多集群生成器
  generators:
    # 华为云国内 prod
    - clusters:
        selector:
          matchLabels:
            cloud: huawei
            env: prod
    # 阿里云海外 prod
    - clusters:
        selector:
          matchLabels:
            cloud: aliyun
            env: prod

  # 统一模板
  template:
    metadata:
      name: 'saas-{{name}}'
      labels:
        app: saas-platform
        cluster: '{{name}}'
    spec:
      project: default

      # 来源: Helm Chart
      source:
        repoURL: https://github.com/company/k8s-config.git
        targetRevision: main
        path: helm/saas-platform
        helm:
          # 每个集群独立配置文件
          valueFiles:
            - values.yaml                    # 默认值
            - values-{{name}}.yaml           # 集群特定
            - values-{{metadata.labels.env}}.yaml  # 环境特定

      destination:
        server: '{{server}}'
        namespace: saas-platform

      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
          - Replace=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m

      # 部署策略
      strategy:
        # 滚动更新 (先华为云、再阿里云)
        rollingSync:
          steps:
            - matchExpressions:
                - key: cloud
                  operator: In
                  values: [huawei]
            - matchExpressions:
                - key: cloud
                  operator: In
                  values: [aliyun]
```

### Step 3: Argo Rollouts 金丝雀发布

```yaml
# Argo Rollout (替代 Deployment)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-gateway
spec:
  replicas: 5
  strategy:
    canary:
      # 金丝雀步骤
      steps:
        - setWeight: 10       # 10% 流量到新版本
        - pause: {duration: 5m}   # 等待 5 分钟
        - setWeight: 30
        - pause: {duration: 10m}
        - setWeight: 60
        - pause: {}
          # 手动确认后继续
        - setWeight: 100

      # 自动分析 (Prometheus 指标)
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1  # 从第二步开始分析

        args:
          - name: service-name
            value: api-gateway
---
# AnalysisTemplate: 自动成功率检查
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 30s
      successCondition: result[0] >= 0.95  # 95% 成功率
      failureLimit: 2                       # 2 次失败即回滚
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(
              http_requests_total{service="{{args.service-name}}",status!~"5.."}[2m]
            )) / sum(rate(
              http_requests_total{service="{{args.service-name}}"}[2m]
            ))
    - name: p95-latency
      interval: 30s
      successCondition: result[0] <= 500   # P95 <500ms
      provider:
        prometheus:
          query: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[2m])) by (le)
            )
```

### Step 4: Helm Chart 模板

```yaml
# helm/saas-platform/templates/deployment.yaml
apiVersion: args/v1
kind: Rollout
metadata:
  name: {{ include "saas-platform.fullname" . }}
  labels: {{- include "saas-platform.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "saas-platform.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "saas-platform.selectorLabels" . | nindent 8 }}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "{{ .Values.metrics.port }}"
    spec:
      serviceAccountName: {{ include "saas-platform.serviceAccountName" . }}
      # 节点亲和 (不同云平台不同策略)
      {{- if eq .Values.cloud "huawei" }}
      nodeSelector:
        node.cce.io/instance-type: "general-computing"
      {{- else if eq .Values.cloud "aliyun" }}
      nodeSelector:
        alibabacloud.com/instance-family: "ecs.g6e"
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: CLOUD_PROVIDER
              value: "{{ .Values.cloud }}"
            - name: REGION
              value: "{{ .Values.region }}"
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
```

```yaml
# helm/saas-platform/values-hcc-cn-prod.yaml (华为云生产环境)
cloud: huawei
region: cn-north-4
replicaCount: 10
resources:
  limits:
    cpu: "4000m"
    memory: "8Gi"
  requests:
    cpu: "2000m"
    memory: "4Gi"
ingress:
  className: cce  # 华为云 CCE Ingress
```

### Step 5: Sealed Secrets (加密敏感配置)

```bash
# 安装 Sealed Secrets Controller (Kubeseal)
# 加密 Secret 后可安全存入 Git
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/latest/controller.yaml

# 创建加密 Secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=Sup3rS3cr3t \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-db-credentials.yaml

# 加密后的 Secret 可安全提交 Git
git add sealed-db-credentials.yaml
git commit -m "Update DB credentials (encrypted)"
```

---

## 💰 成本估算

| 组件 | 月费 |
|------|------|
| Hub ArgoCD (ACK节点) | ¥500 |
| 远程集群 ArgoCD (2节点) | ¥1,000 |
| Argo Rollouts Analyzer (Prometheus存储) | ¥400 |
| **合计** | **~¥1,900** |

---

## 📝 面试常见问题

**Q1: ArgoCD 和 FluxCD 的区别？**
> ArgoCD 有强大的 WebUI + RBAC + ApplicationSet + Rollouts；FluxCD 更轻量、K8s-native (CRD)。企业选 ArgoCD 居多。

**Q2: 多集群 GitOps 怎么做配置差异？**
> ApplicationSet + values-{cluster}.yaml，或 Kustomize overlays。每个集群只需维护一个补丁文件。

**Q3: Sealed Secrets 和 External Secrets 的区别？**
> Sealed Secrets: 加密后存 Git，集群内解密；External Secrets: 从外部 Secret Manager (Vault/AWS SM) 同步，不存 Git。
