# 🚀 企业级云原生架构实战项目

> **设计理念**: 不是"在云虚拟机上装软件"，而是**基于云托管服务构建企业架构**。
> 覆盖阿里云、AWS、腾讯云、华为云、天翼云等主流云平台。

---

## 📋 学习路线图

```
                        ┌─────────────────────────────────────────┐
                        │          🎯 企业云架构师能力模型          │
                        └─────────────────────────────────────────┘
                                              │
        ┌─────────────┬─────────────┬─────────┼─────────┬─────────────┬─────────────┐
        ▼             ▼             ▼         ▼         ▼             ▼             ▼
   Phase 1        Phase 2       Phase 3   Phase 4   Phase 5      Phase 6      Phase 7
  基础设施       容器平台      应用现代化  数据平台  自动化运维    安全合规      进阶专题
  (2项目)       (2项目)       (2项目)    (2项目)   (2项目)      (2项目)       (3项目)
```

---

## 📚 项目总览

### 🔰 Phase 1: 云基础设施 (Cloud Foundation)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 01 | [多云基础设施即代码](./01-terraform-multicloud/README.md) | 阿里云+AWS | ⭐⭐⭐ | Terraform, IaC, 多云管理 |
| 02 | [企业混合云网络架构](./02-vpc-hybrid-cloud/README.md) | 阿里云+华为云 | ⭐⭐⭐⭐ | VPC, 专线, SD-WAN, 混合云 |

### 🐳 Phase 2: 容器与编排 (Container & Orchestration)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 03 | [阿里云 ACK 企业级容器平台](./03-ack-platform/README.md) | 阿里云 | ⭐⭐⭐⭐ | ACK, Terway, 多可用区, 混合节点 |
| 04 | [AWS EKS 多环境 GitOps 交付](./04-aws-eks-gitops/README.md) | AWS | ⭐⭐⭐⭐ | EKS, IRSA, Karpenter, GitOps |

### 🏗️ Phase 3: 应用现代化 (App Modernization)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 05 | [云原生微服务架构](./05-microservices-cloud/README.md) | 阿里云 | ⭐⭐⭐⭐⭐ | MSE, SAE, RocketMQ, 服务网格 |
| 06 | [Serverless 事件驱动架构](./06-serverless/README.md) | AWS+腾讯云 | ⭐⭐⭐⭐ | Lambda, SCF, EventBridge, Step Functions |

### 💾 Phase 4: 数据与存储 (Data & Storage)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 07 | [多云数据库高可用架构](./07-multicloud-db/README.md) | 阿里云+AWS | ⭐⭐⭐⭐ | PolarDB, Aurora, Redis Cluster, DTS |
| 08 | [企业级数据湖与大数据平台](./08-data-lake/README.md) | AWS+阿里云 | ⭐⭐⭐⭐⭐ | S3, OSS, EMR, MaxCompute, Flink |

### ⚙️ Phase 5: 自动化运维 (DevOps & Automation)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 09 | [基于 ArgoCD 多集群 GitOps](./09-gitops-argocd/README.md) | 华为云+阿里云 | ⭐⭐⭐⭐ | ArgoCD, CCE, ACK, Helm |
| 10 | [全链路可观测性平台](./10-observability/README.md) | 阿里云+AWS | ⭐⭐⭐⭐⭐ | SLS, ARMS, CloudWatch, OpenTelemetry |

### 🔒 Phase 6: 安全与治理 (Security & Governance)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 11 | [云安全纵深防御体系](./11-cloud-security/README.md) | 阿里云+天翼云 | ⭐⭐⭐⭐ | WAF, CSPM, KMS, 态势感知, 等保 |
| 12 | [FinOps 多云成本管理](./12-finops/README.md) | 阿里云+AWS+腾讯云 | ⭐⭐⭐ | 成本分析, 资源优化, 预算管理 |

### 🚀 Phase 7: 进阶专题 (Advanced)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 13 | [多云灾备与异地多活](./13-dr-architecture/README.md) | 阿里云+AWS | ⭐⭐⭐⭐⭐ | 异地多活, 数据同步, 流量调度 |
| 14 | [全球边缘计算与CDN](./14-edge-cdn/README.md) | 阿里云+Cloudflare | ⭐⭐⭐⭐ | CDN, EdgeRoutine, DCDN, GA |
| 15 | [企业级 AI/ML 平台架构](./15-ai-ml-platform/README.md) | AWS+阿里云 | ⭐⭐⭐⭐⭐ | SageMaker, PAI, GPU集群, MLOps |

### 🔧 Phase 8: 高级运维专题 (New! 2026-05-01)

| # | 项目 | 云平台 | 难度 | 核心技能 |
|---|------|--------|------|----------|
| 16 | [Service Mesh 网格化治理](./16-service-mesh/README.md) | 阿里云ASM+K8s | ⭐⭐⭐⭐⭐ | Istio, ASM, mTLS, 金丝雀发布 |
| 17 | [DevSecOps 安全左移流水线](./17-devsecops-pipeline/README.md) | GitLab+Trivy | ⭐⭐⭐⭐ | SonarQube, OPA/Kyverno, SBOM |
| 18 | [多云数据库容灾方案](./18-dr-architecture-db/README.md) | 阿里云+华为云 | ⭐⭐⭐⭐⭐ | RDS主备, PolarDB, DTS迁移 |
| 19 | [容器存储 CSI 多厂商适配](./19-csi-storage/README.md) | 阿里云+K8s | ⭐⭐⭐⭐ | CSI Plugin, Dynamic Provisioning, VolumeSnapshot |
| 20 | [弹性伸缩实战 HPA/VPA/KEDA](./20-auto-scaling/README.md) | ACK EHPA | ⭐⭐⭐⭐⭐ | KEDA, Cluster Autoscaler, AHPA |
| 21 | [全链路可观测性体系](./21-observability-stack/README.md) | OTel+SLS | ⭐⭐⭐⭐⭐ | OpenTelemetry, SLS, Grafana, Jaeger |
| 22 | [统一身份管理与SSO](./22-identity-sso/README.md) | Keycloak+RAM | ⭐⭐⭐⭐ | OIDC/SAML, RBAC, Webhook AuthN |
| 23 | [API网关与服务编排](./23-api-gateway/README.md) | APISIX+阿里云 | ⭐⭐⭐⭐ | APISIX, 金丝雀路由, Lua插件 |
| 24 | [混合云GPU算力调度池](./24-gpu-compute-pool/README.md) | 阿里云+AWS | ⭐⭐⭐⭐⭐ | Volcano, NVIDIA Operator, KServe |
| 25 | [配置中心与密钥管理](./25-config-secrets-management/README.md) | Nacos+Vault | ⭐⭐⭐⭐ | Nacos, Vault Agent, SealedSecrets |

---

## 🛠️ 技术栈全景图

```
┌────────────────────────────────────────────────────────────────────────┐
│                          企业云架构技术栈                                │
├──────────────┬──────────────┬──────────────┬──────────────┬─────────────┤
│   阿里云      │     AWS       │   腾讯云      │   华为云      │   天翼云     │
├──────────────┼──────────────┼──────────────┼──────────────┼─────────────┤
│ ACK (K8s)    │ EKS (K8s)    │ TKE (K8s)    │ CCE (K8s)    │ CT-CCE      │
│ SAE (PaaS)   │ ECS/Fargate   │ SCF/Lambda   │ ServiceStage │ 云容器引擎   │
│ MSE (微服务)  │ App Mesh      │ TSF (微服务)  │ CSE (微服务)  │             │
│ PolarDB/RDS  │ Aurora/RDS    │ TDSQL/CynosDB │ GaussDB      │             │
│ Redis 企业版  │ ElastiCache   │ TencentDB     │ DCS          │ 分布式缓存   │
│ RocketMQ     │ SQS/SNS/MSK   │ CKafka        │ DMS/RocketMQ │             │
│ OSS          │ S3            │ COS           │ OBS          │ OOS         │
│ SLS/ARMS     │ CloudWatch/X  │ CLS           │ LTS/AOM      │             │
│ 云效/Flow    │ CodePipeline  │ CODING        │ DevCloud      │             │
│ WAF/DDoS     │ WAF/Shield    │ WAF           │ WAF/AAD      │             │
│ RAM/STS      │ IAM/SSO       │ CAM           │ IAM          │             │
└──────────────┴──────────────┴──────────────┴──────────────┴─────────────┘
```

---

## 🎯 学习建议

### 按角色选择路线

**云基础设施工程师 (Cloud Infra)**
```
01 → 02 → 07 → 11 → 13
```

**DevOps/SRE 工程师**
```
01 → 03/04 → 09 → 10 → 12
```

**云原生开发工程师**
```
03 → 05 → 06 → 08 → 14
```

**云架构师 (全栈)**
```
全部 01→15 按顺序
```

### 实验环境建议

| 方案 | 成本 | 适用场景 |
|------|------|----------|
| **阿里云免费试用** | ¥0 (3个月) | ACK/SAE/RDS 等项目 |
| **AWS Free Tier** | $0 (12个月) | EKS/Lambda/S3 等项目 |
| **华为云开发者计划** | ¥0 | CCE/GaussDB 等项目 |
| **Minikube + LocalStack** | ¥0 (本地) | Terraform/K8s YAML 练习 |

---

## 📝 项目文档结构

每个项目包含:

```
项目名称/
├── README.md          # 项目概述、架构图、实施步骤
├── terraform/         # IaC 代码
├── kubernetes/        # K8s 部署清单
├── pipelines/         # CI/CD Pipeline 配置
├── configs/           # 应用配置
└── docs/              # 详细文档
```

---

## ⚡ 快速开始

```bash
# 1. 安装必要工具
# Terraform + kubectl + helm + aws-cli + aliyun-cli

# 2. 克隆项目
cd /root/.openclaw/workspace/cloud-projects

# 3. 从项目01开始
cd 01-terraform-multicloud
cat README.md
```

---

> **最后更新**: 2026-05-01
> **作者**: ClawOps ⚙️
> **状态**: 🚧 持续更新中
