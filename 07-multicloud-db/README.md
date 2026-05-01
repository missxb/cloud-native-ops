# 项目07：多云数据库高可用架构

> **云平台**: 阿里云 + AWS | **难度**: ⭐⭐⭐⭐ | **工具**: PolarDB, Aurora, Redis, DTS, DMS  
> **场景**: 金融级数据库，国内 PolarDB 主库 + AWS Aurora 灾备，Redis 多活同步

---

## 📖 项目概述

### 业务场景
跨境电商核心交易系统，数据不能丢也不能慢。国内主库阿里云 PolarDB (兼容 MySQL 8.0)，AWS Aurora 做跨云灾备 (DTS 实时同步)。Redis 全球多活实现跨区域会话共享。

### 学习目标
- ✅ PolarDB 集群版架构 (一主多读)
- ✅ Aurora Serverless v2 自动伸缩
- ✅ DTS 跨云实时数据同步
- ✅ Redis 全球多活 (Global Distributed Cache)
- ✅ 数据库审计 + 备份策略

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                    金融级多云数据库架构                                │
│                                                                      │
│  ┌──────────────────────────────────┐ ┌─────────────────────────┐  │
│  │     阿里云 (杭州) - 主力区域       │ │   AWS (俄勒冈) - 灾备    │  │
│  │                                  │ │                         │  │
│  │  ┌────────────────────────┐     │ │  ┌───────────────────┐  │  │
│  │  │   PolarDB 集群          │     │ │  │  Aurora Global DB │  │  │
│  │  │  ┌──────┐ ┌──────┐     │     │ │  │  ┌──────┐ ┌──────┐│  │  │
│  │  │  │读写  │→│只读1 │     │ DTS │ │  │  │ 主   │→│只读  ││  │  │
│  │  │  │节点  │ │节点  │◄────┼─────┼─┼─►│  │ 实例  │ │实例  ││  │  │
│  │  │  └──────┘ ├──────┤     │同步 │ │  │  └──────┘ └──────┘│  │  │
│  │  │           │只读2 │     │     │ │  └───────────────────┘  │  │
│  │  │           └──────┘     │     │ │                         │  │
│  │  └────────────────────────┘     │ │  ┌───────────────────┐  │  │
│  │                                  │ │  │ Redis Global      │  │  │
│  │  ┌────────────────────────┐     │ │  │ Distributed Cache │  │  │
│  │  │   Redis 全球多活        │     │ │  │ (灾备实例)        │  │  │
│  │  │  (主实例 + 同步链路)    │     │ │  └───────────────────┘  │  │
│  │  └────────┬───────────────┘     │ │                         │  │
│  │           │                      │ │                         │  │
│  │  ┌────────┴───────────────┐     │ │  ┌───────┐ ┌───────┐  │  │
│  │  │  DTS 数据同步任务       │     │ │  │Cloud- │ │Route53│  │  │
│  │  │  (全量+增量)            │     │ │  │Watch  │ │故障切 │  │  │
│  │  └────────────────────────┘     │ │  │       │ │换DNS  │  │  │
│  └──────────────────────────────────┘ │  └───────┘ └───────┘  │  │
│                                       └─────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                   DMS (数据管理) 统一入口                   │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: PolarDB 集群创建

```hcl
# PolarDB MySQL 8.0 集群 (一主两读)
resource "alicloud_polardb_cluster" "main" {
  db_type           = "MySQL"
  db_version        = "8.0"
  db_node_class     = "polar.mysql.x4.xlarge"  # 8C32G 独占
  pay_type          = "PostPaid"
  vswitch_id        = alicloud_vswitch.private_azg.id
  description       = "ecommerce-main-db"

  # 删除保护
  deletion_lock     = 1
  
  # 存储: 按量付费自动扩展
  storage_pay_type  = "PostPaid"
  
  # 维护时间
  maintain_time     = "04:00Z-05:00Z"
  
  # 安全
  tde_status        = "Enabled"
  tde_region        = "cn-hangzhou"
  
  # 参数配置
  parameters {
    name  = "max_connections"
    value = "2000"
  }
  parameters {
    name  = "innodb_buffer_pool_size"
    value = "{DBInstanceClassMemory*7/10}"
  }
}

# 创建只读节点
resource "alicloud_polardb_cluster_endpoint" "readonly" {
  db_cluster_id = alicloud_polardb_cluster.main.id
  endpoint_type = "ReadOnly"          # 自动读写分离
  read_write_mode = "ReadOnly"
  auto_add_new_nodes = "Enable"       # 新节点自动加入
  
  nodes {
    node_id = alicloud_polardb_cluster.main.db_nodes[1].node_id
    weight  = 1
  }
  nodes {
    node_id = alicloud_polardb_cluster.main.db_nodes[2].node_id
    weight  = 1
  }
}

# 数据库账号 (最小权限)
resource "alicloud_polardb_account" "app" {
  db_cluster_id       = alicloud_polardb_cluster.main.id
  account_name        = "app_user"
  account_password    = random_password.db.result
  account_type        = "Normal"
  account_description = "Application service account"
}

# 数据库权限
resource "alicloud_polardb_account_privilege" "app" {
  db_cluster_id     = alicloud_polardb_cluster.main.id
  account_name      = alicloud_polardb_account.app.account_name
  db_names          = ["ecommerce"]
  account_privilege = "ReadWrite"
}
```

### Step 2: DTS 跨云同步 (阿里云 → AWS)

```hcl
# DTS 同步任务: PolarDB → Aurora
resource "alicloud_dts_synchronization_instance" "cross_cloud" {
  payment_type                     = "PayAsYouGo"
  source_endpoint_engine_name      = "MySQL"
  destination_endpoint_engine_name = "MySQL"
  
  # 同步拓扑: 单向 (one way)
  topology = "oneway"
  
  # 同步网络: 公网 (跨云) + SSL
  network_type = "public"
  
  # 同步规格
  synchronization_direction = "Forward"
  class                     = "large"  # 支持较大同步流量
}

# DTS 同步任务配置
resource "alicloud_dts_synchronization_job" "main" {
  dts_instance_id                    = alicloud_dts_synchronization_instance.cross_cloud.id
  dts_job_name                       = "polar-to-aurora-sync"
  data_initialization                = true   # 全量同步
  data_synchronization               = true   # 增量同步
  structure_initialization           = true   # 结构同步
  source_endpoint_instance_type      = "PolarDB"
  source_endpoint_instance_id        = alicloud_polardb_cluster.main.id
  source_endpoint_region             = "cn-hangzhou"
  source_endpoint_user_name          = alicloud_polardb_account.dts.account_name
  source_endpoint_password           = random_password.dts.result
  destination_endpoint_instance_type = "others"
  destination_endpoint_instance_id   = aws_rds_cluster.aurora.id
  destination_endpoint_region        = "us-west-2"
  destination_endpoint_user_name     = aws_rds_cluster.aurora.master_username
  destination_endpoint_password      = aws_rds_cluster.aurora.master_password
  
  # 同步对象 (只同步业务库)
  db_list = jsonencode({
    ecommerce = {
      name = "ecommerce"
      all  = true
    }
  })
}
```

### Step 3: Aurora Global Database (AWS 侧)

```hcl
# Aurora MySQL 8.0 Serverless v2 (灾备)
module "aurora" {
  source  = "terraform-aws-modules/rds-aurora/aws"
  version = "~> 8.0"

  name              = "ecommerce-dr"
  engine            = "aurora-mysql"
  engine_version    = "8.0.mysql_aurora.3.05"
  engine_mode       = "provisioned"  # 非 serverless，因 Global DB 要求
  
  vpc_id            = module.vpc.vpc_id
  subnets           = module.vpc.database_subnets
  
  # 集群配置
  master_username   = "admin"
  manage_master_user_password = true  # 自动密钥管理
  database_name     = "ecommerce"
  
  instances = {
    writer = {
      instance_class = "db.r6g.xlarge"
      publicly_accessible = false
    }
    reader = {
      instance_class = "db.r6g.xlarge"
      publicly_accessible = false
    }
  }
  
  # 备份
  backup_retention_period = 30    # 30天
  preferred_backup_window = "06:00-07:00"  # UTC
  
  # 安全
  iam_database_authentication_enabled = true
  storage_encrypted                    = true
  kms_key_id                           = aws_kms_key.rds.arn
  deletion_protection                  = true
  
  # Deletion Protection 等
  enabled_cloudwatch_logs_exports = ["audit", "error", "general", "slowquery"]
  copy_tags_to_snapshot           = true
}
```

### Step 4: Redis 全球多活

```hcl
# 阿里云 Redis 全球分布式缓存
resource "alicloud_kvstore_instance" "main" {
  db_instance_name  = "ecommerce-redis-main"
  instance_class    = "redis.logic.sharding.4g.4db.0db_4c_24g_instance"
  engine_version    = "7.0"
  instance_type     = "Redis"
  vswitch_id        = alicloud_vswitch.private_azg.id
  security_ips      = ["172.16.0.0/16"]
  
  # 全球多活: 主实例
  global_instance   = true
  global_instance_id = alicloud_kvstore_global_instance.main.id
  
  # 配置
  config = {
    maxmemory-policy = "allkeys-lru"
    timeout          = "300"
  }
  
  # 自动备份
  backup_id = data.alicloud_kvstore_backup_policy.main.id
}

# AWS 侧 ElastiCache 灾备 (AOF 持久化)
resource "aws_elasticache_replication_group" "dr" {
  replication_group_id = "ecommerce-cache-dr"
  description          = "Redis DR instance"
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.xlarge"
  num_cache_clusters   = 2
  port                 = 6379
  subnet_group_name    = aws_elasticache_subnet_group.dr.name
  security_group_ids   = [aws_security_group.redis.id]
  
  automatic_failover_enabled   = true
  multi_az_enabled             = true
  at_rest_encryption_enabled   = true
  transit_encryption_enabled   = true
  auth_token                   = random_password.redis_auth.result
  
  # 参数组
  parameter_group_name = "default.redis7.cluster.on"
}

# 自定义参数: AOF 持久化
resource "aws_elasticache_parameter_group" "custom" {
  name   = "ecommerce-redis7"
  family = "redis7"
  
  parameter {
    name  = "appendonly"
    value = "yes"
  }
  parameter {
    name  = "appendfsync"
    value = "everysec"
  }
}
```

### Step 5: 备份策略

```yaml
# 备份策略矩阵
backup_matrix:
  polar:
    full_backup: "daily 03:00, retain 14 days"
    binlog: "realtime, retain 7 days"
    clone: "PolarDB 秒级克隆到新集群"
    
  aurora:
    continuous: "Aurora Backtrack 72h (秒级回滚)"
    snapshot: "daily, retain 30 days"
    export: "weekly to S3 Glacier"
    
  redis:
    rdb: "daily 03:00, retain 7 days"
    aof: "everysec (AWS side)"
```

---

## 📊 关键 SLA 指标

| 指标 | PolarDB | Aurora |
|------|---------|--------|
| 可用性 | 99.99% | 99.99% |
| RPO (数据丢失) | <1秒 (Binlog) | <5秒 (跨云DTS) |
| RTO (恢复时间) | <5分钟 | <15分钟 |
| 读扩展 | 最多16个只读节点 | 最多15个只读副本 |
| 存储上限 | 100TB | 128TiB |

---

## 💰 成本估算

| 资源 | 规格 | 月费 |
|------|------|------|
| PolarDB | 8C32G+2只读 | ~¥18,000 |
| Aurora | db.r6g.xlarge×2 | ~$800 |
| DTS 同步 | large, 跨云 | ~¥4,500 |
| Redis 阿里云 | 4C24G 4分片 | ~¥3,500 |
| ElastiCache | r6g.xlarge×2 | ~$380 |
| **合计** | | **~¥29,000 + ~$1,180** |

---

## 📝 面试常见问题

**Q1: PolarDB 和 Aurora 技术架构有什么区别？**
> 相似：都是存算分离架构，日志即数据库。PolarDB 用 PolarFS 分布式共享存储，Aurora 用分布式存储层。PolarDB Serverless 支持秒级弹性，Aurora Serverless v2 支持毫秒级。

**Q2: DTS 跨云同步延迟怎么监控？**
> DTS 提供源延迟 + 目标延迟双指标，可在 ARMS/CloudWatch 配置告警。正常延迟 <5秒。

**Q3: 数据库灾备切换流程？**
> 1. 确认主库不可用；2. 停止 DTS 同步链路；3. 提升 Aurora 只读实例为独立集群；4. DNS 切换流量；5. 反向同步激活。

---

## 📚 扩展阅读

- [PolarDB 架构原理](https://help.aliyun.com/document_detail/58768.html)
- [Aurora 技术内幕](https://aws.amazon.com/rds/aurora/)
- [DTS 跨云同步](https://help.aliyun.com/document_detail/264569.html)
