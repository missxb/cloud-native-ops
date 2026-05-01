# 项目01：多云基础设施即代码 (Multi-Cloud IaC)

> **云平台**: 阿里云 + AWS | **难度**: ⭐⭐⭐ | **工具**: Terraform, Terragrunt, Atlantis  
> **场景**: 中大型企业需要在阿里云和 AWS 同时管理基础设施，要求代码化、版本化、自动化

---

## 📖 项目概述

### 业务场景
某跨境电商公司业务遍布全球，国内主要使用阿里云，海外使用 AWS。运维团队只有5人，需要管理上百台云资源。通过 Terraform 实现基础设施即代码 (IaC)，将资源管理从"控制台点点点"变成"代码 Review + CI/CD 自动执行"。

### 学习目标
- ✅ 掌握 Terraform HCL 语法与模块化设计
- ✅ 理解 Terraform State 管理与远程 Backend
- ✅ 实现多云资源统一管理
- ✅ 学会 Terragrunt DRY 配置
- ✅ 集成 Atlantis 实现 PR 驱动的 IaC 工作流

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitOps IaC Pipeline                          │
│                                                                      │
│   Developer PR ──► GitHub/GitLab ──► Atlantis ──► Terraform Plan    │
│                                                    │                │
│                                              Review & Approve        │
│                                                    │                │
│                                              Terraform Apply         │
│                                                    │                │
│                          ┌─────────────────────────┼────────────┐   │
│                          ▼                         ▼            ▼   │
│                    ┌──────────┐            ┌──────────┐            │
│                    │  阿里云    │            │   AWS     │            │
│                    ├──────────┤            ├──────────┤            │
│                    │ VPC      │            │ VPC      │            │
│                    │ ECS      │            │ EC2      │            │
│                    │ SLB      │            │ ALB      │            │
│                    │ RDS      │            │ RDS      │            │
│                    │ Redis    │            │ Elastic..│            │
│                    │ OSS      │            │ S3       │            │
│                    │ RAM      │            │ IAM      │            │
│                    └──────────┘            └──────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📂 项目结构

```
01-terraform-multicloud/
├── README.md
├── terraform/
│   ├── modules/                    # 可复用模块
│   │   ├── vpc/                    # VPC 通用模块
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── ecs-ec2/               # 计算实例模块 (多云适配)
│   │   ├── rds/                   # 数据库模块
│   │   ├── redis/                 # 缓存模块
│   │   └── security-group/        # 安全组模块
│   ├── aliyun/                    # 阿里云环境
│   │   ├── prod/
│   │   │   ├── terragrunt.hcl     # Terragrunt 配置
│   │   │   ├── main.tf
│   │   │   └── terraform.tfvars
│   │   └── staging/
│   │       └── ...
│   ├── aws/                       # AWS 环境
│   │   ├── prod/
│   │   └── staging/
│   └── global/                    # 全局资源 (IAM/RAM, DNS)
│       └── ...
├── atlantis/
│   └── atlantis.yaml              # Atlantis 工作流配置
├── scripts/
│   ├── setup-backend.sh           # 初始化 OSS/S3 Backend
│   └── import-existing.sh         # 现有资源导入脚本
└── docs/
    ├── state-management.md        # State 管理最佳实践
    └── multi-cloud-patterns.md    # 多云设计模式
```

---

## 🔧 实施步骤

### Step 1: 初始化 Terraform Backend

**阿里云 Backend (OSS)**

```hcl
# aliyun/prod/backend.tf
terraform {
  backend "oss" {
    bucket              = "tfstate-prod-bucket"
    key                 = "aliyun/prod/terraform.tfstate"
    region              = "cn-hangzhou"
    access_key          = var.aliyun_access_key
    secret_key          = var.aliyun_secret_key
    encrypt             = true
    acl                 = "private"
  }
}
```

**AWS Backend (S3 + DynamoDB Lock)**

```hcl
# aws/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "tfstate-prod-bucket"
    key            = "aws/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Step 2: 编写 VPC 模块

```hcl
# modules/vpc/main.tf
resource "alicloud_vpc" "main" {
  count       = var.cloud_provider == "aliyun" ? 1 : 0
  vpc_name    = "${var.environment}-${var.name}"
  cidr_block  = var.cidr_block
}

resource "aws_vpc" "main" {
  count       = var.cloud_provider == "aws" ? 1 : 0
  cidr_block  = var.cidr_block
  
  tags = {
    Name        = "${var.environment}-${var.name}"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# 阿里云交换机
resource "alicloud_vswitch" "private" {
  count             = var.cloud_provider == "aliyun" ? length(var.private_subnets) : 0
  vpc_id            = alicloud_vpc.main[0].id
  cidr_block        = var.private_subnets[count.index]
  zone_id           = var.availability_zones[count.index]
  vswitch_name      = "${var.environment}-private-${count.index}"
}

# AWS 私有子网 (带 NAT Gateway 路由自动关联)
resource "aws_subnet" "private" {
  count             = var.cloud_provider == "aws" ? length(var.private_subnets) : 0
  vpc_id            = aws_vpc.main[0].id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.environment}-private-${count.index}"
  }
}
```

### Step 3: Terragrunt 实现 DRY

```hcl
# aliyun/prod/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "../../../modules//vpc"
}

inputs = {
  cloud_provider     = "aliyun"
  environment        = "prod"
  name               = "ecommerce"
  region             = "cn-hangzhou"
  cidr_block         = "10.0.0.0/16"
  private_subnets    = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets     = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  availability_zones = ["cn-hangzhou-g", "cn-hangzhou-h", "cn-hangzhou-i"]
}
```

### Step 4: Atlantis GitOps 工作流

```yaml
# atlantis/atlantis.yaml
version: 3
projects:
  - name: aliyun-prod
    dir: terraform/aliyun/prod
    workspace: default
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "terragrunt.hcl"]
      enabled: true
  - name: aws-prod
    dir: terraform/aws/prod
    workspace: default
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true

workflows:
  terragrunt:
    plan:
      steps:
        - run: terragrunt plan -out=$PLANFILE
    apply:
      steps:
        - run: terragrunt apply $PLANFILE
```

### Step 5: CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/terraform.yml
name: "Terraform CI/CD"
on:
  pull_request:
    paths:
      - 'terraform/**'
  push:
    branches: [main]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        
      - name: Terraform Validate (Aliyun)
        working-directory: terraform/aliyun/prod
        run: terraform init -backend=false && terraform validate
      
      - name: Terraform Validate (AWS)
        working-directory: terraform/aws/prod
        run: terraform init -backend=false && terraform validate
      
      - name: TFSec Security Scan
        uses: aquasecurity/tfsec-action@v1.0.0
```

---

## 🎓 关键知识

### Terraform State 管理黄金法则

| 原则 | 说明 | 反模式 |
|------|------|--------|
| **1 State = 1 环境** | 每个环境独立 State 文件 | 所有环境共用一个 State |
| **远程存储** | OSS/S3/Consul，不存本地 | git 提交 .tfstate |
| **状态锁** | DynamoDB/Consul 防止并发 | 多人同时 apply |
| **加密** | State 文件加密存储 | 明文存储含密钥的 State |
| **最小权限** | 用不同 Service Account | 一个账号管理所有环境 |

### 多云设计模式

```
┌────────────────────────────────────────────┐
│           Abstraction Layer                 │
│   ┌─────────────────────────────────┐      │
│   │   Provider-Agnostic Interface    │      │
│   │   (modules/vpc, modules/db...)   │      │
│   └───────────┬─────────────────────┘      │
│               │                             │
│   ┌───────────┴───────────┐                │
│   │   Provider Adapters    │                │
│   ├───────────┬───────────┤                │
│   │  Aliyun   │    AWS    │                │
│   │  Adapter  │  Adapter  │                │
│   └───────────┴───────────┘                │
└────────────────────────────────────────────┘
```

### 安全最佳实践

1. **敏感变量管理**: 使用 `terraform.tfvars` + `.gitignore`，或 HashiCorp Vault
2. **最小权限**: 阿里云 RAM 策略、AWS IAM Policy 遵循最小权限原则
3. **State 文件不泄露**: `.gitignore` 排除 `.tfstate*`
4. **安全扫描**: tfsec / checkov 集成到 CI
5. **资源标签强制**: 通过 Policy 强制资源打标，方便审计和成本追踪

---

## 💰 成本估算

| 资源 | 阿里云 (月) | AWS (月) |
|------|------------|----------|
| Terraform Backend (OSS/S3) | ~¥5 | ~$0.5 |
| Atlantis 服务器 (2C4G) | ~¥200 | ~$30 |
| CI/CD Runner | 免费 (GitHub Actions) | 免费 |
| 测试 VPC (无 NAT) | ¥0 | $0 |
| **总计** | **~¥205** | **~$30.5** |

> ⚠️ 仅 Atlantis 服务器有成本，其他为免费或极低成本

---

## 📝 面试常见问题

**Q1: Terraform 和 Pulumi/Crossplane 有什么区别？**
> Terraform 是声明式 IaC DSL(领域特定语言)，Pulumi 用通用编程语言(Golang/Python等)写IaC，Crossplane 是 K8s-native IaC (用K8s CRD管理云资源)。选型看团队技术栈。

**Q2: State 文件损坏怎么恢复？**
> 从远程 Backend 的版本历史恢复，或从 `terraform refresh` 重新同步。阿里云OSS有版本控制，AWS S3也有。关键是要启用版本控制。

**Q3: 如何管理多环境(dev/staging/prod)？**
> Terragrunt + 目录隔离，或 Terraform Workspace。推荐前者，因为隔离更彻底。

**Q4: `terraform destroy` 误操作怎么办？**
> 1. 启用 `prevent_destroy` lifecycle；2. 开启删除保护(云平台)；3. 通过 Atlantis 要求多级审批。

---

## 📚 扩展阅读

- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/)
- [Atlantis Documentation](https://www.runatlantis.io/)
- [阿里云 Terraform Provider](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs)
- [AWS Terraform Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
