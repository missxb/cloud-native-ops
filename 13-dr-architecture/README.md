# 项目13：多云灾备与异地多活

> **云平台**: 阿里云 + AWS | **难度**: ⭐⭐⭐⭐⭐ | **工具**: DTS, Route53, GTN, DCDN, DNS 智能调度  
> **场景**: 双11级电商需异地多活架构，任一 Region 故障不影响业务

---

## 📖 项目概述

电商平台需要支持 Region 级别故障切换。构建阿里云(杭州) + AWS(新加坡) 异地多活架构，RTO<1分钟，RPO<5秒。

### 学习目标
- ✅ 异地多活架构设计原则
- ✅ 多活流量调度 (DNS + GTN)
- ✅ 跨云数据双向同步冲突处理
- ✅ 单元化架构 (流量路由规则)
- ✅ 容灾演练 (混沌工程)

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      异地多活架构 (Active-Active)                     │
│                                                                      │
│  用户 → ┌────────────────────────────────────────┐                  │
│        │       全局流量管理 (GTN / Route53)        │                  │
│        │   • 地理近源路由 (Geo DNS)               │                  │
│        │   • 健康检查 (Health Check)              │                  │
│        │   • 故障自动切换 Failover               │                  │
│        └──────┬──────────────────┬───────────────┘                  │
│               │                  │                                   │
│      ┌────────┴───────┐  ┌───────┴──────────┐                      │
│      │ Region A (杭州) │  │ Region B (新加坡) │                      │
│      │ 阿里云 100%     │  │ AWS 100%         │                      │
│      ├────────────────┤  ├──────────────────┤                      │
│      │ ┌────────────┐ │  │ ┌──────────────┐ │                      │
│      │ │ CDN/DCDN   │ │  │ │ CloudFront   │ │                      │
│      │ └─────┬──────┘ │  │ └──────┬───────┘ │                      │
│      │       │        │  │        │         │                      │
│      │ ┌─────┴──────┐ │  │ ┌──────┴───────┐ │                      │
│      │ │   SLB/NLB  │ │  │ │  ALB/NLB     │ │                      │
│      │ └─────┬──────┘ │  │ └──────┬───────┘ │                      │
│      │       │        │  │        │         │                      │
│      │ ┌─────┴──────┐ │  │ ┌──────┴───────┐ │                      │
│      │ │ ACK 集群    │ │  │ │ EKS 集群     │ │                      │
│      │ │ 微服务集群  │ │  │ │ 微服务集群   │ │                      │
│      │ └─────┬──────┘ │  │ └──────┬───────┘ │                      │
│      │       │        │  │        │         │                      │
│      │ ┌─────┴──────┐ │  │ ┌──────┴───────┐ │                      │
│      │ │PolarDB(写) │◄┼──┼►│Aurora(读)    │ │                      │
│      │ │Redis+MQ    │ │  │ │Cache+SQS     │ │                      │
│      │ └────────────┘ │  │ └──────────────┘ │                      │
│      └────────────────┘  └──────────────────┘                      │
│               │                  │                                   │
│      ┌────────┴──────────────────┴──────┐                           │
│      │      DTS 双向同步 + 冲突解决      │                           │
│      │      (全局递增 ID / CRDT)         │                           │
│      └──────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: 全局流量调度 (GTM)

```hcl
# 阿里云全局流量管理 (GTM)
resource "alicloud_gtm_instance" "multi_region" {
  instance_name   = "ecommerce-multi-region"
  payment_type    = "Subscription"
  package_edition = "standard"
  health_check_task_count = 2
  sms_notification_count   = 1000
}

# 地址池: 杭州集群
resource "alicloud_gtm_address_pool" "hangzhou" {
  instance_id   = alicloud_gtm_instance.multi_region.id
  name          = "hangzhou-pool"
  type          = "IP"
  
  address {
    mode    = "SMART"
    lba_strategy = "ALL_RR"
    address {
      value     = slb_hangzhou_ip
      type      = "IPv4"
    }
  }
}

# 地址池: 新加坡集群
resource "alicloud_gtm_address_pool" "singapore" {
  instance_id   = alicloud_gtm_instance.multi_region.id
  name          = "singapore-pool"
  type          = "IP"
  address {
    address {
      value     = alb_singapore_dns
      type      = "IPv4"
    }
  }
}

# 访问策略: 智能解析
resource "alicloud_gtm_access_strategy" "multi_region" {
  instance_id        = alicloud_gtm_instance.multi_region.id
  strategy_name      = "geo-routing"
  default_addr_pool_id = alicloud_gtm_address_pool.hangzhou.id
  strategy_mode      = "GEO"
  
  # 解析线路配置
  lines {
    line_code = "default"
  }
  lines {
    line_code = "cn"
    group_code = "CHINA"
    addr_pool_id = alicloud_gtm_address_pool.hangzhou.id
  }
  lines {
    line_code = "sg"
    group_code = "ASIA_PACIFIC"
    addr_pool_id = alicloud_gtm_address_pool.singapore.id
  }
}

# AWS Route53 多活 DNS (AWS 侧)
resource "aws_route53_health_check" "singapore" {
  fqdn              = "api-sg.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
  
  regions = ["ap-southeast-1", "us-west-2", "eu-west-1"]
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  
  # 地理位置路由
  set_identifier = "hangzhou"
  geolocation_routing_policy {
    continent = "Asia"
  }
  alias {
    name                   = "slb-hangzhou.aliyuncs.com"
    zone_id                = "Z1ANCZBK9BEYFB"  # 阿里云 SLB zone id
    evaluate_target_health = true
  }
  
  # 新加坡故障切换
  failover_routing_policy {
    type = "PRIMARY"
  }
  health_check_id = aws_route53_health_check.singapore.id
}
```

### Step 2: 单元化流量路由

```yaml
# 单元化路由配置
# 用户按 userId 哈希分配到固定单元 (Region)
# Nginx / Spring Cloud Gateway 层实现

# Nginx 配置: 单元化路由
# nginx.conf
map $http_x_user_id $backend_region {
    # hash(userId) % 100  → 决定路由到哪个单元
    # 0-49: 杭州  50-99: 新加坡
    ~^(\d*[05])$    "hz";           # 尾号0/5 → 杭州
    ~^(\d*[2])$     "hz";           # 尾号2   → 杭州
    ~^(\d*[8])$     "hz";           # 尾号8   → 杭州
    # ...
    default         "sg";           # 其余 → 新加坡
}

upstream hangzhou_upstream {
    server hz-gateway.internal:8080;
}
upstream singapore_upstream {
    server sg-gateway.internal:8080;
}

server {
    location /api/ {
        # 单元化路由
        if ($backend_region = "hz") {
            proxy_pass http://hangzhou_upstream;
        }
        if ($backend_region = "sg") {
            proxy_pass http://singapore_upstream;
        }
        proxy_set_header X-Region $backend_region;
    }
}
```

### Step 3: 数据双向同步 + 冲突解决

```sql
-- 多活数据库设计: 全局递增 ID 避免冲突
-- 方案1: Snowflake 变体 (WorkerID 编码 Region)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    -- id 编码: 41-bit timestamp + 10-bit worker(region+dc) + 12-bit sequence
    -- Region A worker = 1-100, Region B worker = 101-200
    -- 天然不冲突

    user_id BIGINT,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    sync_version INT DEFAULT 0  -- 乐观锁版本号
);

-- 冲突检测: 基于版本号
-- DTS 同步到对端时检查 sync_version
-- 如果 target.version > source.version → 保留 target (更新的)
-- 如果 target.version = source.version → 按 Region 优先级仲裁

-- 双向同步规则
-- PolarDB (杭州) ← DTS → Aurora (新加坡)
-- 仅同步: orders, users, products 表
-- 排除: sessions, logs, cache 表 (本地生成)
```

### Step 4: 故障切换 SOP

```yaml
# 容灾切换剧本 (SOP)
failover_playbook:
  # 场景1: Region A (杭州) 完全故障
  scenario_hz_down:
    detection: "GTM 健康检查 3次失败"
    automated:
      - step: "DNS 自动切换 → 全部流量指到新加坡"
        tool: "GTM / Route53 Failover"
        target_rto: "< 60s"
      - step: "CDN 源站切换 (若CDN回源杭州)"
        tool: "DCDN 源站修改"
        target_rto: "< 5min"
    manual:
      - step: "确认 Aurora 提升为独立可写 (断开DTS)"
        rpo: "< 5s (DTS同步延迟)"
      - step: "通知运营/客服团队"
      - step: "监控新加坡集群容量: 触发紧急扩容"
    
  # 场景2: DTS 同步链路中断
  scenario_dts_down:
    detection: "DTS 监控告警 (延迟 > 30s)"
    automated:
      - step: "暂停受影响的双向写入单元"
      - step: "告警通知 DBA 值班"
    manual:
      - step: "评估数据差异量"
      - step: "修复链路 or 重新全量同步"

  # 演练计划
  chaos_engineering:
    - monthly: "DNS 故障切换演练 (非高峰期)"
    - quarterly: "Region 级全链路压测 + 切换演练"
    - annually: "双11前全栈容灾大演练"
```

### Step 5: 混沌工程 (LitmusChaos)

```yaml
# LitmusChaos 实验: 模拟 Region 故障
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: region-failure-experiment
spec:
  engineState: "active"
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-loss
      spec:
        components:
          env:
            # 模拟网络分区: 中断跨 Region 通信
            - name: TARGET_CONTAINER
              value: "cross-region-sync"
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_PACKET_LOSS_PERCENTAGE
              value: "100"
            - name: TOTAL_CHAOS_DURATION
              value: "120"  # 2 分钟
        probe:
          - name: "check-failover-triggered"
            type: "cmdProbe"
            cmd:
              command: "./verify_gtm_failover.sh"
            mode: "EOT"
            runProperties:
              probeTimeout: 30
              retry: 3
              interval: 10
```

---

## 📊 多活 vs 主备对比

| 维度 | 主备 (Active-Passive) | 多活 (Active-Active) |
|------|----------------------|---------------------|
| 资源利用率 | 50% (备机闲置) | ~90% (双活) |
| 故障切换时间 | 15-30分钟 | <1分钟 (DNS) |
| 数据一致性 | 强一致 (备库readonly) | 最终一致 (双向同步) |
| 架构复杂度 | 低 | 高 (需要冲突解决) |
| 写入扩展 | 单点写入 | 双写 (按单元路由) |
| 适合场景 | 金融核心交易 | 电商/社交/游戏 |

---

## 💰 成本估算 (多活 vs 单 Region)

| 项目 | 单 Region | 多活 (2 Region) |
|------|-----------|-----------------|
| 计算 | ¥10,000 | ¥18,000 |
| 数据库 | ¥15,000 | ¥28,000 (双 DB) |
| 网络 (专线/DTS) | 0 | ¥8,000 |
| 流量管理 (GTM) | 0 | ¥3,000 |
| **合计** | **¥25,000** | **¥57,000** |

> 多活架构增加 ~130% 成本，但换来 99.99%+ 可用性

---

## 📝 面试常见问题

**Q1: 异地多活最大技术难点？**
> 数据一致性与冲突解决。方案: 单元化(用户维度分片) + 全局递增ID + CRDT 数据类型 + 业务层补偿。

**Q2: 为什么不用数据库自带的多活 (如 Aurora Global DB)？**
> Aurora Global DB 仍是主备模式(1写多读, 跨Region只读)。真正的多活需要双写能力。

**Q3: 如何验证容灾方案有效性？**
> 混沌工程 (Chaos Engineering) 定期演练。从DNS切换→DB提升→应用层全链路验证。
