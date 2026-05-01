# 项目08：企业级数据湖与大数据平台

> **云平台**: AWS + 阿里云 | **难度**: ⭐⭐⭐⭐⭐ | **工具**: S3, OSS, EMR, MaxCompute, Flink, Hudi  
> **场景**: 电商数据平台，日均 5TB 增量数据，准实时 + 离线分析，支持 BI + AI 双场景

---

## 📖 项目概述

某电商平台需要统一分析用户行为、交易、日志数据，支撑实时大屏、BI 报表、用户画像、推荐系统。采用 Lakehouse 架构 (数据湖 + 数仓融合)，开源 + 托管混合。

### 学习目标
- ✅ Lakehouse 架构设计 (S3/OSS + Hudi/Iceberg)
- ✅ EMR Serverless / MaxCompute 批处理
- ✅ Flink 流计算 (Flink SQL + CDC)
- ✅ 数据血缘、治理、权限管控
- ✅ BI 对接 (Quick BI / QuickSight)

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Lakehouse 数据平台架构                            │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │                     数据源层                                │      │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │      │
│  │  │ RDS日志│ │ 埋点   │ │ IOT数据│ │ 业务DB │ │ 第三方 │ │      │
│  │  │        │ │ Kafka  │ │        │ │(CDC)   │ │ API    │ │      │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ │      │
│  └──────┼──────────┼──────────┼──────────┼──────────┼──────┘      │
│         │          │          │          │          │              │
│  ┌──────┴──────────┴──────────┴──────────┴──────────┴──────┐      │
│  │                     数据接入层 (Flink CDC/Kafka Connect)  │      │
│  └──────────────────┬──────────────────────────────────────┘      │
│                     │                                               │
│  ┌──────────────────┴──────────────────────────────────────┐      │
│  │                    数据存储层 (Lakehouse)                  │      │
│  │                                                           │      │
│  │   Bronze (原始)  →  Silver (清洗)  →  Gold (聚合)        │      │
│  │   S3/OSS          S3/OSS + Hudi     S3/OSS + Hudi        │      │
│  │   JSON/CSV        Parquet+ZOrder    Star Schema           │      │
│  │                    (CDC实时同步)     (按小时聚合)          │      │
│  └──────────────────┬──────────────────────────────────────┘      │
│                     │                                               │
│  ┌──────────────────┴──────────────────────────────────────┐      │
│  │                    计算引擎                                │      │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │      │
│  │  │ 流处理 Flink │ │ 批处理 Spark │ │ 交互 Trino    │     │      │
│  │  │ (EMR/实时    │ │(EMR/MaxComp) │ │ (Presto/Athena│     │      │
│  │  │ 计算 Flink)  │ │              │ │ 即席查询)     │     │      │
│  │  └──────────────┘ └──────────────┘ └──────────────┘     │      │
│  └──────────────────┬──────────────────────────────────────┘      │
│                     │                                               │
│  ┌──────────────────┴──────────────────────────────────────┐      │
│  │                    服务层                                  │      │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │      │
│  │  │ BI 报表  │ │ 实时大屏 │ │ 用户画像 │ │ 推荐模型 │   │      │
│  │  │Quick BI  │ │DataV/GV  │ │  + 标签  │ │  + 特征  │   │      │
│  │  │QuickSight│ │Grafana   │ │ 平台     │ │ 工程     │   │      │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: 数据存储层 (S3/OSS + Hudi)

```hcl
# 三层存储 Bucket
# Bronze: 原始数据, 7天过期
resource "aws_s3_bucket" "bronze" {
  bucket = "ecommerce-data-lake-bronze"
  
  lifecycle_rule {
    id      = "expire-raw"
    enabled = true
    expiration { days = 7 }
  }
}

# Silver: 清洗后数据, 90天
resource "aws_s3_bucket" "silver" {
  bucket = "ecommerce-data-lake-silver"
  lifecycle_rule = {
    id      = "transition-to-gb"
    enabled = true
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    expiration { days = 90 }
  }
}

# Gold: 聚合数据, 永久保留
resource "aws_s3_bucket" "gold" {
  bucket = "ecommerce-data-lake-gold"
  versioning { enabled = true }
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm     = "aws:kms"
        kms_master_key_id = aws_kms_key.data_lake.arn
      }
    }
  }
}
```

### Step 2: Flink CDC 实时同步 (MySQL → Hudi)

```sql
-- Flink SQL: CDC 实时同步订单表到数据湖
-- 使用 Flink CDC Connector + Hudi Connector

-- Step 1: 创建源表 (MySQL CDC)
CREATE TABLE orders_source (
    order_id    BIGINT PRIMARY KEY NOT ENFORCED,
    user_id     BIGINT,
    product_id  BIGINT,
    amount      DECIMAL(10,2),
    status      STRING,
    created_at  TIMESTAMP(3),
    updated_at  TIMESTAMP(3)
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'polar-master.rds.aliyuncs.com',
    'port' = '3306',
    'username' = 'flink_cdc_user',
    'password' = '${SECRET}',
    'database-name' = 'ecommerce',
    'table-name' = 'orders',
    'server-time-zone' = 'Asia/Shanghai'
);

-- Step 2: 创建目标表 (Apache Hudi on S3/OSS)
CREATE TABLE orders_silver (
    order_id    BIGINT,
    user_id     BIGINT,
    product_id  BIGINT,
    amount      DECIMAL(10,2),
    status      STRING,
    created_at  TIMESTAMP(3),
    updated_at  TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'hudi',
    'path' = 's3://ecommerce-data-lake-silver/orders/',
    'table.type' = 'MERGE_ON_READ',   -- MOR 读时合并
    'write.operation' = 'upsert',
    'hoodie.datasource.write.recordkey.field' = 'order_id',
    'hoodie.datasource.write.precombine.field' = 'updated_at',
    'compaction.async.enabled' = 'true',
    'compaction.schedule.enabled' = 'true'
);

-- Step 3: 实时同步
INSERT INTO orders_silver SELECT * FROM orders_source;
```

### Step 3: EMR Serverless 批处理 (Spark)

```python
# PySpark Job on EMR Serverless (阿里云)
# 天级聚合: Silver → Gold

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder \
    .appName("DailyAggregation") \
    .config("spark.sql.extensions", 
            "org.apache.spark.sql.hudi.HoodieSparkSessionExtension") \
    .getOrCreate()

# 读取 Silver 层订单数据 (Hudi MOR 表)
orders = spark.read.format("hudi") \
    .load("oss://ecommerce-data-lake-silver/orders/")

# 天级聚合: 按商品+状态统计
daily_agg = orders \
    .withColumn("dt", to_date(col("created_at"))) \
    .groupBy("dt", "product_id", "status") \
    .agg(
        count("order_id").alias("order_count"),
        sum("amount").alias("total_amount"),
        avg("amount").alias("avg_amount"),
    )

# 写入 Gold 层
daily_agg.write \
    .format("hudi") \
    .option("hoodie.datasource.write.recordkey.field", "dt,product_id,status") \
    .option("hoodie.datasource.write.precombine.field", "dt") \
    .option("hoodie.datasource.write.operation", "bulk_insert") \
    .option("hoodie.table.name", "daily_order_agg") \
    .mode("append") \
    .save("oss://ecommerce-data-lake-gold/daily_order_agg/")
```

### Step 4: MaxCompute 数仓 (阿里云)

```sql
-- 阿里云 MaxCompute (ODPS) + DataWorks 定时调度
-- 创建数仓分层表 (DWD → DWS → ADS)

-- DWD 层: 订单明细事实表
CREATE TABLE IF NOT EXISTS dwd_order_detail (
    order_id        BIGINT,
    user_id         BIGINT,
    product_id      BIGINT,
    product_name    STRING,
    category_id     BIGINT,
    category_name   STRING,
    amount          DECIMAL(10,2),
    status          STRING,
    dt              STRING  -- 分区: yyyyMMdd
) PARTITIONED BY (dt);

-- 每天凌晨通过 DataWorks 调度执行
ALTER TABLE dwd_order_detail ADD IF NOT EXISTS PARTITION (dt='${bizdate}');
INSERT OVERWRITE TABLE dwd_order_detail PARTITION (dt='${bizdate}')
SELECT 
    o.order_id, o.user_id, o.product_id,
    p.product_name, p.category_id, c.category_name,
    o.amount, o.status
FROM ods_order o
LEFT JOIN dim_product p ON o.product_id = p.product_id
LEFT JOIN dim_category c ON p.category_id = c.category_id
WHERE o.dt = '${bizdate}';
```

### Step 5: 数据治理 (Data Catalog + Lineage)

```hcl
# AWS Glue Data Catalog
resource "aws_glue_catalog_database" "data_lake" {
  name        = "ecommerce_data_lake"
  description = "Ecommerce Lakehouse catalog"
}

# Glue Crawler 自动发现 Schema
resource "aws_glue_crawler" "silver" {
  name          = "ecommerce-silver-crawler"
  database_name = aws_glue_catalog_database.data_lake.name
  role          = aws_iam_role.glue.arn

  s3_target {
    path = "s3://ecommerce-data-lake-silver/"
  }

  schema_change_policy {
    delete_behavior = "LOG"
    update_behavior = "UPDATE_IN_DATABASE"
  }
}

# 阿里云 DataWorks 数据血缘
# 自动追踪表间依赖, 可视化 ETL 流程
# 控制台 → DataWorks → 数据地图 → 血缘分析
```

---

## 📊 技术选型矩阵

| 场景 | AWS 方案 | 阿里云方案 | 开源组件 |
|------|----------|-----------|----------|
| 数据湖存储 | S3 + Lake Formation | OSS + DLF | MinIO + Hive |
| Table Format | Apache Iceberg | Apache Hudi/Iceberg | Hudi/Iceberg/Delta |
| 流处理 | Kinesis Analytics (Flink) | 实时计算 Flink | Apache Flink |
| 批处理 | EMR (Spark) | MaxCompute / EMR | Apache Spark |
| 交互查询 | Athena (Trino) | MaxCompute SQL | Trino/Presto |
| 调度 | Step Functions + EventBridge | DataWorks | Airflow/DolphinScheduler |
| 数据治理 | Glue + DataZone | DataWorks | Amundsen/DataHub |

---

## 💰 成本估算

| 服务 | 月费 |
|------|------|
| S3/OSS 存储 (50TB) | ~$1,150 + ~¥4,200 |
| EMR Serverless (1100 vCPU·h) | ~$200 |
| MaxCompute (按量) | ~¥3,000 |
| Flink 实时计算 (16CU) | ~¥2,800 |
| DataWorks 调度 | ~¥2,000 |
| Glue Crawler + Catalog | ~$30 |
| **合计** | **~$1,380 + ~¥12,000** |

---

## 📝 面试常见问题

**Q1: Hudi/Iceberg/Delta Lake 怎么选？**
> Hudi: Upsert 最强 + 增量查询；Iceberg: 生态广泛 (Netflix/AWS 力推)；Delta: Databricks 生态。AWS 推荐 Iceberg。

**Q2: 数据湖和数据仓库的区别？**
> 数据湖存储原始格式，支持结构化+非结构化，成本低；数仓是预处理后的结构化数据，查询性能高。Lakehouse 融合两者。

**Q3: 如何保证实时同步的数据一致性？**
> Flink CDC + Checkpoint 机制保证 Exactly-Once；Hudi 的事务日志保证写入原子性。
