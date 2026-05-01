# 项目05：云原生微服务架构

> **云平台**: 阿里云 | **难度**: ⭐⭐⭐⭐⭐ | **工具**: MSE, SAE, RocketMQ, ASM (服务网格), Seata  
> **场景**: 从 Spring Cloud 自建微服务迁移到阿里云全托管微服务体系

---

## 📖 项目概述

### 业务场景
某企业原有 50+ Spring Cloud 微服务，自建 Nacos + Sentinel + Seata + RocketMQ，运维成本高。迁移到阿里云 MSE(注册配置中心) + SAE(Serverless应用引擎) + ASM(服务网格)，实现托管化。

### 学习目标
- ✅ 掌握 MSE Nacos/Sentinel/Seata 托管版
- ✅ SAE 实现 Spring Cloud 应用零改造上云
- ✅ ASM 服务网格实现流量治理与可观测性
- ✅ RocketMQ 5.0 消息驱动解耦
- ✅ 微服务全链路灰度发布

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                    阿里云全托管微服务架构                               │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │                    流量入口 (SLB + API Gateway)              │      │
│  └────────────────────────┬──────────────────────────────────┘      │
│                           │                                         │
│  ┌────────────────────────┴──────────────────────────────────┐      │
│  │              ASM 服务网格 (Istio 托管版)                      │      │
│  │   ┌─────────────────────────────────────────────────┐     │      │
│  │   │  流量治理: 灰度/限流/熔断/负载均衡                │     │      │
│  │   │  可观测: Tracing/Metrics/Logging 统一采集         │     │      │
│  │   │  安全: mTLS + JWT + RBAC                        │     │      │
│  │   └─────────────────────────────────────────────────┘     │      │
│  └────────────────────────┬──────────────────────────────────┘      │
│                           │                                         │
│  ┌────────────────────────┴──────────────────────────────────┐      │
│  │              SAE (Serverless 应用引擎)                       │      │
│  │                                                            │      │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │      │
│  │  │ 订单服务 │ │ 用户服务 │ │ 支付服务 │ │ 库存服务 │ ...    │      │
│  │  │4C8G×3   │ │2C4G×3  │ │4C8G×3  │ │2C4G×3  │        │      │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    中间件托管                               │      │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │      │
│  │  │   MSE    │ │  MSE     │ │ RocketMQ │ │  Seata   │    │      │
│  │  │ Nacos    │ │ Sentinel │ │   5.0    │ │ (MSE版)  │    │      │
│  │  │(注册配置) │ │(Sentinel) │ │(消息队列) │ │(分布式事务)│   │      │
│  │  │          │ │           │ │          │ │          │    │      │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: MSE 全托管中间件

```hcl
# MSE Nacos (注册配置中心)
resource "alicloud_mse_cluster" "nacos" {
  cluster_specification = "MSE_SC_2_4_200_c"  # 2C4G × 3节点
  cluster_type          = "Nacos-Ans"
  cluster_version       = "NACOS_2_3_0"
  instance_count        = 3
  net_type              = "privatenet"
  pub_network_flow      = "0"  # 仅内网访问
  acl_entry_list        = ["172.16.0.0/16"]
  vswitch_id            = alicloud_vswitch.private_azg.id
  vpc_id                = alicloud_vpc.main.id
}

# MSE Sentinel (流量防护)
resource "alicloud_mse_cluster" "sentinel" {
  cluster_specification = "MSE_SC_1_2_200_c"
  cluster_type          = "Sentinel-App"
  cluster_version       = "SENTINEL_1_6_3"
  instance_count        = 1
  net_type              = "privatenet"
  vswitch_id            = alicloud_vswitch.private_azg.id
  vpc_id                = alicloud_vpc.main.id
}

# RocketMQ 5.0 实例
resource "alicloud_ons_instance" "main" {
  instance_name = "ecommerce-rocketmq"
  series_code   = "standard"  # 标准版 (5.0)
  msg_retain    = 72          # 消息保留 72 小时
  remark        = "电商业务主MQ"
}

# 创建 Topic
resource "alicloud_ons_topic" "order_created" {
  instance_id = alicloud_ons_instance.main.id
  topic       = "ORDER_CREATED"
  message_type = 0  # 普通消息
  remark      = "订单创建事件"
}

# 创建 Consumer Group
resource "alicloud_ons_group" "inventory_consumer" {
  instance_id = alicloud_ons_instance.main.id
  group_id    = "GID_INVENTORY"
  remark      = "库存服务消费组"
}
```

### Step 2: Spring Boot 应用改造

```yaml
# bootstrap.properties - 零代码改动
spring.application.name=order-service
spring.profiles.active=prod

# MSE Nacos 地址 (不需要自建)
spring.cloud.nacos.discovery.server-addr=mse-xxx-nacos.mse.aliyuncs.com:8848
spring.cloud.nacos.config.server-addr=mse-xxx-nacos.mse.aliyuncs.com:8848
spring.cloud.nacos.config.namespace=prod
spring.cloud.nacos.config.file-extension=yaml

# MSE Sentinel 地址
spring.cloud.sentinel.transport.dashboard=mse-xxx-sentinel.mse.aliyuncs.com:8080

# RocketMQ 5.0 Producer
rocketmq.name-server=http://MQ_INST_xxx.cn-hangzhou.mq-internal.aliyuncs.com:8080
rocketmq.producer.group=GID_ORDER_SERVICE
```

### Step 3: SAE 部署配置

```yaml
# SAE 应用配置 (通过 CLI 或控制台)
apiVersion: sae.alibabacloud.com/v1
kind: Application
metadata:
  name: order-service
  namespace: prod
spec:
  # 基础配置
  replicas: 3
  cpu: "2000m"      # 2 Core
  memory: "4096Mi"  # 4 GiB
  vpcId: vpc-xxx
  vSwitchId: vswitch-xxx
  
  # 镜像或 Jar 包
  packageType: Image
  imageUrl: registry.cn-hangzhou.aliyuncs.com/ecommerce/order-service:v2.5.0
  
  # Java 启动参数
  commandArgs: >
    -Xms2048m -Xmx2048m
    -XX:+UseG1GC
    -Dfile.encoding=UTF-8
    --spring.profiles.active=prod
  
  # 弹性策略
  autoScale:
    minReplicas: 3
    maxReplicas: 20
    metrics:
      cpu: 70
      memory: 80
    
  # 启动健康检查
  livenessProbe:
    tcpSocket:
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    
  readinessProbe:
    httpGet:
      path: /actuator/health
      port: 8080
    initialDelaySeconds: 20
    periodSeconds: 5
  
  # 挂载 MSE Config
  envs:
    - name: JAVA_TOOL_OPTIONS
      value: >-
        -Dspring.cloud.nacos.discovery.server-addr=mse-xxx-nacos.mse.aliyuncs.com:8848
    
  # 关联 SLS 日志
  slsConfigs:
    - logDir: /home/admin/logs
      logstoreName: order-service-logs
```

### Step 4: ASM 服务网格流量治理

```yaml
# ASM VirtualService - 灰度发布
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: prod
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            version:
              exact: v2.5.0
      route:
        - destination:
            host: order-service
            subset: v250  # 新版本
      # 新版本专用超时配置
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 2s
    - route:
        - destination:
            host: order-service
            subset: v240
          weight: 95     # 95% 流量老版本
        - destination:
            host: order-service
            subset: v250
          weight: 5      # 5% 流量新版本 (金丝雀)
---
# Destination Rule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: prod
spec:
  host: order-service
  # mTLS 自动加密
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    # 连接池限流
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    # 异常检测
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v240
      labels:
        version: v2.4.0
    - name: v250
      labels:
        version: v2.5.0
```

### Step 5: 分布式事务 (Seata)

```yaml
# 订单创建流程: 订单→库存→积分 (跨服务事务)
# Seata AT 模式 (MSE 提供)

spring:
  cloud:
    alibaba:
      seata:
        tx-service-group: ecommerce-tx-group

# application.yml
seata:
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: mse-xxx-nacos.mse.aliyuncs.com:8848
      namespace: prod
      group: SEATA_GROUP
  config:
    type: nacos
    nacos:
      server-addr: mse-xxx-nacos.mse.aliyuncs.com:8848
      namespace: prod
      group: SEATA_GROUP
      data-id: seata.properties
```

```java
// 业务代码只需加注解
@GlobalTransactional(timeoutMills = 300000)
public Order createOrder(OrderDTO dto) {
    // 1. 创建订单
    Order order = orderRepository.save(dto.toOrder());
    
    // 2. 扣减库存 (远程调用, 自动加入事务)
    inventoryService.decreaseStock(dto.getProductId(), dto.getQuantity());
    
    // 3. 增加积分 (远程调用)
    pointsService.addPoints(dto.getUserId(), calculatePoints(dto));
    
    // 任何一步失败 → Seata 自动回滚全部
    return order;
}
```

---

## 📊 迁移前后对比

| 维度 | 自建微服务 | 阿里云托管 |
|------|-----------|-----------|
| Nacos 高可用 | 手动搭3节点 | MSE自动3AZ |
| Sentinel | 手动部署控制台 | MSE 托管 |
| RocketMQ | 手动搭集群 | 按量/包年包月 |
| 服务网格 | 自建 Istio | ASM 托管 |
| 灰度发布 | 手动切换 | ASM + MSE 一键 |
| 弹性伸缩 | HPA (分钟级) | SAE 自带 (秒级) |
| 运维工作 | 全天候值班 | 99.95% SLA |

---

## 💰 成本估算

| 服务 | 月费 (¥) |
|------|----------|
| MSE Nacos (2C4G×3) | ¥2,700 |
| MSE Sentinel | ¥900 |
| RocketMQ Standard | ¥3,000 |
| ASM (50节点) | ¥7,500 |
| SAE 应用 (总8C16G) | ¥3,200 |
| **合计** | **~¥17,300** |

---

## 📝 面试常见问题

**Q1: SAE 和 ACK 有什么区别？**
> SAE 是 Serverless 应用引擎，不需要管理 K8s 节点，直接部署 Jar/War/Image，适合 Spring Cloud 应用。ACK 是完整 K8s 集群，更灵活。

**Q2: 服务网格 Sidecar 有什么开销？**
> Envoy Sidecar 每个 Pod 额外消耗 ~50m CPU + 50MB 内存；ASM 托管版免去 Istiod 控制面运维。

**Q3: Seata AT 和 TCC 模式怎么选？**
> AT 模式对业务无侵入(自动生成undo log)，适合标准CRUD；TCC 需要业务实现 Try/Confirm/Cancel 接口，性能更好。

---

## 📚 扩展阅读

- [SAE 产品文档](https://help.aliyun.com/product/113549.html)
- [MSE 微服务引擎](https://help.aliyun.com/product/123350.html)
- [ASM 服务网格](https://help.aliyun.com/product/123251.html)
