# 项目04：AWS EKS 多环境 GitOps 交付

> **云平台**: AWS | **难度**: ⭐⭐⭐⭐ | **工具**: EKS, Karpenter, ArgoCD, IRSA, Crossplane  
> **场景**: SaaS 公司需要管理 dev/staging/prod 三套 EKS 集群，通过 GitOps 实现多环境一致性交付

---

## 📖 项目概述

### 业务场景
某 SaaS 企业向全球客户提供服务，使用 3 套 EKS 集群 (US-East/Europe/APAC)。通过 GitOps (ArgoCD + Karpenter) 实现应用和基础设施的声明式管理。

### 学习目标
- ✅ EKS 集群 Terraform 自动化创建
- ✅ Karpenter 智能节点调度 (替代 Cluster Autoscaler)
- ✅ IRSA 实现 Pod 级 IAM 权限
- ✅ ArgoCD ApplicationSet 多集群管理
- ✅ External Secrets Operator + AWS Secrets Manager

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      GitOps 驱动 DevOps 流水线                         │
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│   │ App Repo │    │ Infra Repo│   │ Config Repo│                    │
│   └────┬─────┘    └────┬─────┘    └────┬──────┘                    │
│        │               │               │                           │
│        └───────────────┼───────────────┘                           │
│                        │ Git Push                                   │
│                        ▼                                            │
│              ┌──────────────────┐                                   │
│              │     ArgoCD        │  ← Multi-Cluster Controller      │
│              │   ApplicationSet  │                                  │
│              └───┬───────┬───────┘                                  │
│                  │       │                                          │
│     ┌────────────┴┐     ┌┴────────────┐     ┌────────────┐         │
│     │ EKS Prod    │     │ EKS Staging │     │ EKS Dev    │         │
│     │ (us-east-1) │     │ (eu-west-1) │     │ (ap-se-1)  │         │
│     ├─────────────┤     ├─────────────┤     ├─────────────┤        │
│     │ Karpenter   │     │ Karpenter   │     │ Karpenter   │        │
│     │ ALB Ingress │     │ ALB Ingress │     │ ALB Ingress │        │
│     │ +────────── │     │ +────────── │     │ +────────── │        │
│     │ Aurora RDS  │     │ (共享 Dev)  │     │             │        │
│     │ ElastiCache │     │             │     │             │        │
│     │ Secrets Mgr │     │ Secrets Mgr │     │ Secrets Mgr │        │
│     └─────────────┘     └─────────────┘     └─────────────┘        │
│                                                                      │
│   ┌────────────────────────────────────────────────────────────┐     │
│   │                    AWS 托管服务层                            │     │
│   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │     │
│   │  │Route53│ │  S3  │ │  ECR │ │SQS/SNS│ │  KMS │ │ WAF  │   │     │
│   │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │     │
│   └────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: Terraform 创建 EKS 集群

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "saas-prod"
  cluster_version = "1.28"

  # 网络: 仅私有子网 (安全)
  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.intra_subnets  # 控制面内网

  # EKS 托管节点组 (初始节点, Karpenter 后续接管)
  eks_managed_node_groups = {
    initial = {
      instance_types = ["m6i.large", "m6a.large", "c6i.large"]
      min_size       = 3
      max_size       = 5
      desired_size   = 3
      
      # 启动模板自定义
      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 100
            volume_type           = "gp3"
            encrypted             = true
            kms_key_id            = aws_kms_key.eks.arn
          }
        }
      }
    }
  }

  # IRSA (IAM Roles for Service Accounts) 预创建
  eks_managed_node_groups_defaults = {
    iam_role_attach_cw_policy = true
    create_iam_role           = true
  }

  tags = {
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

# EKS Blueprints Addons (一键安装常用插件)
module "eks_blueprints_addons" {
  source = "aws-ia/eks-blueprints-addons/aws"

  cluster_name      = module.eks.cluster_name
  cluster_endpoint  = module.eks.cluster_endpoint
  cluster_version   = module.eks.cluster_version
  oidc_provider_arn = module.eks.oidc_provider_arn

  enable_karpenter                  = true
  enable_aws_load_balancer_controller = true
  enable_external_dns                = true
  enable_cert_manager                = true
  enable_external_secrets            = true
  enable_aws_efs_csi_driver          = true
  enable_metrics_server              = true

  karpenter = {
    repository_username = data.aws_ecrpublic_authorization_token.token.user_name
    repository_password = data.aws_ecrpublic_authorization_token.token.password
  }
}
```

### Step 2: Karpenter 智能节点调度

```yaml
# Karpenter NodePool - 通用计算
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 1m
---
# Karpenter NodePool - GPU 工作负载
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["g", "p"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["large", "xlarge", "2xlarge"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: gpu
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
---
# EC2NodeClass 配置
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: "KarpenterNodeRole-saas-prod"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "saas-prod"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "saas-prod"
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
  metadataOptions:
    httpEndpoint: enabled
    httpTokens: required    # IMDSv2 强制
```

### Step 3: ArgoCD ApplicationSet 多集群

```yaml
# ApplicationSet: 根据集群标签自动部署
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    # 集群生成器: 匹配标签 env=prod 的集群
    - clusters:
        selector:
          matchLabels:
            env: prod
    # 同时匹配 staging 集群 (叠加生成)
    - clusters:
        selector:
          matchLabels:
            env: staging
  template:
    metadata:
      name: '{{name}}-microservices'
      labels:
        environment: '{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/microservices.git
        targetRevision: '{{metadata.annotations.appRev}}'
        path: helm/microservices
        helm:
          valueFiles:
            - 'values.yaml'
            - 'values-{{name}}.yaml'  # 环境特定配置
      destination:
        server: '{{server}}'
        namespace: microservices
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: false
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

### Step 4: External Secrets Operator

```yaml
# SecretStore 定义从哪里读取密钥
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
---
# ExternalSecret: 声明式同步 AWS 密钥到 K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: prod/rds/master-credentials  # AWS Secrets Manager key
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/rds/master-credentials
        property: password
```

### Step 5: ALB Ingress 自动配置

```yaml
# AWS Load Balancer Controller 自动创建 ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/waf-acl-id: xxx-arid
    # 多可用区
    alb.ingress.kubernetes.io/subnets: subnet-xxx,subnet-yyy,subnet-zzz
    # 安全组自动创建
    alb.ingress.kubernetes.io/security-groups: sg-xxx
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders/*
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
          - path: /users/*
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
```

---

## 📊 EKS vs ACK 关键对比

| 特性 | AWS EKS | 阿里云 ACK |
|------|---------|------------|
| CNI 插件 | VPC CNI (独立ENI) | Terway (类似) |
| 服务账号 | IRSA (临时凭证) | RRSA (临时凭证) |
| 弹性伸缩 | Karpenter 推荐 | AHPA + ECI |
| Ingress | ALB Controller | ALB Ingress (ack-ingress-nginx) |
| 密钥管理 | External Secrets + SM | ack-secret-manager + KMS |
| 多集群 | EKS Connector | ACK One |

---

## 💰 成本估算

| 资源 | 月费 |
|------|------|
| EKS 控制面 | $73 |
| 节点 (3×m6i.large) | ~$210 |
| ALB | ~$25 |
| Secrets Manager | ~$3 |
| ECR 镜像仓库 | ~$5 |
| **合计** | **~$316/月** |

---

## 📝 面试常见问题

**Q1: Karpenter 和 Cluster Autoscaler 的区别？**
> Karpenter 可以直接创建节点无需 ASG，支持多机型混合、装箱优化、consolidation (碎片整理)。CA 必须依赖 ASG。

**Q2: IRSA 的实现原理？**
> EKS 集群有 OIDC Provider ↔ IAM OIDC Identity Provider。通过 ServiceAccount 注解关联 IAM Role，STS 交换临时凭证。

**Q3: ArgoCD ApplicationSet 解决了什么问题？**
> 不用为每个环境/集群重复创建 Application。通过生成器 (cluster/list/git) 自动生成，DRY 原则。

---

## 📚 扩展阅读

- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [Karpenter Documentation](https://karpenter.sh/)
- [ArgoCD ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
