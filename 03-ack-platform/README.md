# 项目03：阿里云 ACK 企业级容器平台

> **云平台**: 阿里云 | **难度**: ⭐⭐⭐⭐ | **工具**: ACK, Terway, AHPA, MSE 微服务引擎  
> **场景**: 电商公司在阿里云构建容器平台支撑双11级流量，要求多可用区高可用、弹性伸缩、混合节点

---

## 📖 项目概述

### 业务场景
某电商平台日均 PV 5000万，大促期间 PV 5亿+。需要容器平台支撑秒级弹性扩容，同时兼顾成本 (部分业务使用 ECI/抢占式实例)。要求99.95%可用性，RTO<5分钟。

### 学习目标
- ✅ 掌握 ACK Pro 版集群创建与配置
- ✅ 理解 Terway 网络插件 (独占ENI vs 共享ENI)
- ✅ 实现 HPA + AHPA 智能弹性伸缩
- ✅ 混合节点池 (ECS + ECI + 抢占式)
- ✅ 集成 MSE 微服务引擎 + 服务网格

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                   阿里云 ACK 企业级容器平台                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                      SLB (公网 + 内网)                       │     │
│  └────────────┬───────────────────────────────┬───────────────┘     │
│               │                               │                     │
│  ┌────────────┴───────────┐    ┌──────────────┴──────────────┐     │
│  │    可用区 G (主)        │    │    可用区 H (备)             │     │
│  │  ┌──────────────────┐  │    │  ┌──────────────────┐      │     │
│  │  │ ECS 节点池        │  │    │  │ ECS 节点池        │      │     │
│  │  │ 3×ecs.g6e.4xlarge│  │    │  │ 3×ecs.g6e.4xlarge│      │     │
│  │  ├──────────────────┤  │    │  ├──────────────────┤      │     │
│  │  │ ECI 虚拟节点      │  │    │  │ ECI 虚拟节点      │      │     │
│  │  │ (突发流量弹性)    │  │    │  │ (突发流量弹性)    │      │     │
│  │  ├──────────────────┤  │    │  ├──────────────────┤      │     │
│  │  │ 抢占式节点池      │  │    │  │ 抢占式节点池      │      │     │
│  │  │ (离线/批处理)    │  │    │  │ (离线/批处理)    │      │     │
│  │  └──────────────────┘  │    │  └──────────────────┘      │     │
│  └────────────────────────┘    └────────────────────────────┘     │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    托管服务 (VPC 内)                         │     │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │     │
│  │  │PolarDB│ │ Redis │ │Rocket│ │ MSE  │ │ OSS  │ │ SLS  │   │     │
│  │  │(主库)│ │企业版│ │  MQ  │ │(注册 │ │(对象 │ │(日志 │   │     │
│  │  │      │ │      │ │      │ │配置)│ │存储)│ │服务)│   │     │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                    可观测性                                  │     │
│  │  ┌──────────────────┐  ┌──────────────────┐                │     │
│  │  │ ARMS Prometheus  │  │    ARMS APM      │                │     │
│  │  │ (指标采集)        │  │  (链路追踪)       │                │     │
│  │  └──────────────────┘  └──────────────────┘                │     │
│  └────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: Terraform 创建 ACK 集群

```hcl
# 创建 ACK Pro 版集群
resource "alicloud_cs_managed_kubernetes" "prod" {
  name                           = "ecommerce-prod"
  cluster_spec                   = "ack.pro.small"
  version                        = "1.28"
  
  # 专有网络配置
  worker_vswitch_ids             = [
    alicloud_vswitch.private_azg.id,
    alicloud_vswitch.private_azh.id,
  ]
  pod_vswitch_ids                = [
    alicloud_vswitch.pod_azg.id,
    alicloud_vswitch.pod_azh.id,
  ]  # Terway 模式下 Pod 独立交换机

  # 网络: Terway (阿里云自研 CNI, 性能优于 Flannel/Calico)
  service_cidr                   = "172.16.0.0/20"
  pod_cidr                       = ""  # Terway 模式下由 VPC 分配
  
  addons {
    name = "terway-eniip"       # 共享 ENI 模式 (IPVLAN)
    # name = "terway-eni"       # 独占 ENI 模式 (性能更高)
  }
  addons {
    name = "arms-prometheus"    # 托管 Prometheus
  }
  addons {
    name = "ack-node-problem-detector"
  }
  addons {
    name = "csi-plugin"         # 云盘/NAS 存储
  }
  addons {
    name = "csi-provisioner"
  }

  # 安全配置
  rrsa_enabled                   = true  # RRSA (RAM Roles for Service Accounts)
  deletion_protection            = true
  resource_group_id              = alicloud_resource_group.prod.id
}

# 默认节点池 (ECS)
resource "alicloud_cs_kubernetes_node_pool" "default" {
  name                          = "default-pool"
  cluster_id                    = alicloud_cs_managed_kubernetes.prod.id
  vswitch_ids                   = [
    alicloud_vswitch.private_azg.id,
    alicloud_vswitch.private_azh.id,
  ]
  
  instance_types                = ["ecs.g6e.4xlarge"]
  system_disk_category          = "cloud_essd"
  system_disk_size              = 120
  system_disk_performance_level = "PL1"
  
  desired_size                  = 6
  max_size                      = 20
  min_size                      = 3
  
  scaling_policy                = "release"
  
  # 自动修复
  auto_repair                   = true
  auto_repair_policy {
    restart_node = true
  }
  
  # 节点标签
  labels {
    key   = "node-type"
    value = "compute"
  }
  
  # taints 防止非关键 Pod 调度
  taints {
    key    = "dedicated"
    value  = "compute"
    effect = "NoSchedule"
  }
  
  # KMS 加密
  kms_encrypted_password = var.node_password
  cis_cis_enabled        = true  # CIS 安全基线
}

# ECI 虚拟节点 (弹性容器实例) - 免运维、按量计费
resource "alicloud_cs_kubernetes_node_pool" "eci" {
  name                          = "elastic-eci-pool"
  cluster_id                    = alicloud_cs_managed_kubernetes.prod.id
  vswitch_ids                   = [alicloud_vswitch.private_azg.id]
  
  instance_types                = ["ecs.eci"]
  desired_size                  = 0
  
  labels {
    key   = "node-type"
    value = "eci"
  }
}

# 抢占式实例节点池 (低优先级/批处理)
resource "alicloud_cs_kubernetes_node_pool" "spot" {
  name                          = "spot-pool"
  cluster_id                    = alicloud_cs_managed_kubernetes.prod.id
  vswitch_ids                   = [alicloud_vswitch.private_azh.id]
  
  instance_types                = [
    "ecs.g6e.4xlarge",
    "ecs.g7.4xlarge",   # 多规格打散, 防止回收
  ]
  instance_charge_type          = "PostPaid"
  spot_strategy                 = "SpotWithPriceLimit"
  spot_price_limit              = [
    { instance_type = "ecs.g6e.4xlarge", price_limit = "2.5" },
    { instance_type = "ecs.g7.4xlarge",  price_limit = "2.5" },
  ]
  desired_size                  = 3
  max_size                      = 10
  
  labels {
    key   = "node-type"
    value = "spot"
  }
  taints {
    key    = "spot"
    value  = "true"
    effect = "NoSchedule"
  }
}
```

### Step 2: 智能弹性伸缩 (AHPA)

```yaml
# AHPA (Advanced HPA) - 预测式弹性
# 结合历史数据预测流量高峰, 提前扩容
apiVersion: autoscaling.alibabacloud.com/v1beta1
kind: AdvancedHorizontalPodAutoscaler
metadata:
  name: order-service-ahpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 5
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  # 预测配置 (AHPA 特有)
  prediction:
    quantile: 0.95          # 95 分位预测
    scaleUpForward: 180     # 提前 180 秒扩容
  # 缩容策略 (避免抖动)
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
  # 弹性边界
  boundary:
    period: "0 8-22 * * *"       # 白天
    minReplicas: 10
    maxReplicas: 100
  boundaryNight:
    period: "0 0-8,22-24 * * *"  # 夜间
    minReplicas: 2
    maxReplicas: 20
```

### Step 3: RRSA (RAM Roles for Service Accounts)

```yaml
# 替代传统 AccessKey/Secret，使用 RAM 角色
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: prod
  annotations:
    ram.alibabacloud.com/role: acs:ram::123456:role/order-service-oss-role
---
# Pod 使用该 SA 即可自动获取临时凭证
apiVersion: v1
kind: Pod
metadata:
  name: order-service
spec:
  serviceAccountName: order-service-sa
  containers:
  - name: app
    image: registry.cn-hangzhou.aliyuncs.com/order:v1.0
    env:
    - name: OSS_ENDPOINT
      value: oss-cn-hangzhou-internal.aliyuncs.com
```

### Step 4: MSE 微服务引擎集成

```yaml
# 通过 MSE Cloud Map 实现服务发现 (替代自建 Nacos)
apiVersion: mse.alibabacloud.com/v1
kind: MSEApplication
metadata:
  name: ecommerce-app
spec:
  serviceDiscovery:
    type: Nacos
    serverAddr: mse-xxx-nacos-ans.mse.aliyuncs.com:8848
  configManagement:
    type: Nacos
    serverAddr: mse-xxx-nacos-ans.mse.aliyuncs.com:8848
---
# Spring Boot 应用配置 (bootstrap.yml)
spring:
  cloud:
    nacos:
      discovery:
        server-addr: mse-xxx-nacos-ans.mse.aliyuncs.com:8848
        namespace: prod
      config:
        server-addr: mse-xxx-nacos-ans.mse.aliyuncs.com:8848
        namespace: prod
        file-extension: yaml
```

### Step 5: 存储 CSI 配置

```yaml
# ESSD 云盘 (高 IOPS, 用于数据库类应用)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-essd
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  type: cloud_essd
  performanceLevel: PL2  # 更高IOPS
  encrypted: "true"
  kmsKeyId: xxx-kms-key-id
allowVolumeExpansion: true
reclaimPolicy: Delete
---
# NAS (共享存储, 用于多 Pod 共享)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-nas
provisioner: nasplugin.csi.alibabacloud.com
parameters:
  volumeAs: subpath
  server: "xxxxxx.cn-hangzhou.nas.aliyuncs.com:/"
  archiveOnDelete: "false"
```

---

## 📊 集群规格对照

| 规格 | 最大节点数 | 最大 Pod 数 | 适用场景 |
|------|-----------|------------|----------|
| ack.pro.small | 100 | 6,000 | 中小业务 |
| ack.pro.medium | 500 | 30,000 | 中型业务 |
| ack.pro.large | 2000 | 120,000 | 大型电商/金融 |

---

## 💰 成本估算 (月度)

| 组件 | 规格 | 月费 (¥) |
|------|------|----------|
| ACK Pro 集群管理 | ack.pro.small | ¥5,400 |
| 默认节点池 | 6×ecs.g6e.4xlarge | ¥12,960 |
| 抢占式节点池 | 3×ecs.g6e.4xlarge | ~¥2,000 |
| SLB 实例 | 2 个 | ¥300 |
| MSE Nacos | 2C4G × 3节点 | ¥2,700 |
| ARMS Prometheus | 托管版 | ¥1,600 |
| **合计** | | **~¥24,960** |

---

## 📝 面试常见问题

**Q1: Terway 和 Calico/Flannel 的区别？**
> Terway 是阿里云自研 CNI，基于 VPC ENI，性能更高、支持 NetworkPolicy、Pod 直接获得 VPC IP (无需 SNAT)。缺点是多云迁移不便。

**Q2: HPA 和 AHPA 有什么区别？**
> HPA 基于实时指标被动伸缩 (滞后性)，AHPA 结合历史数据预测 (提前扩容) + 自动降级到 HPA。

**Q3: RRSA 解决了什么问题？**
> 不用在 Pod 里配 AccessKey/Secret，通过 ServiceAccount 绑定 RAM 角色获取临时凭证，安全且自动轮转。

**Q4: ECI 和普通 ECS 节点的区别？**
> ECI 是 Serverless 容器实例，无需管理节点、按秒计费，适合突发流量。缺点是冷启动延迟(30s)和价格较高。

---

## 📚 扩展阅读

- [ACK Pro 集群文档](https://help.aliyun.com/document_detail/184065.html)
- [Terway 网络模式](https://help.aliyun.com/document_detail/97467.html)
- [AHPA 预测式弹性](https://help.aliyun.com/document_detail/434946.html)
- [MSE 微服务引擎](https://help.aliyun.com/product/123350.html)
