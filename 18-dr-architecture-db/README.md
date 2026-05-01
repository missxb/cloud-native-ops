# 项目十八: 多云数据库高可用同城容灾方案

## 一、项目背景

企业生产数据库面临的关键挑战：
- **单点故障**: 数据库服务器宕机导致业务中断
- **数据中心级灾难**: 机房断电/火灾/地震等不可逆故障
- **数据一致性**: 跨地域同步延迟导致读写不一致
- **合规要求**: 金融/医疗等行业要求 RPO≤5分钟、RTO≤30分钟

容灾目标：
| 指标 | 定义 | 目标值 |
|------|------|--------|
| RPO (Recovery Point Objective) | 最大允许丢失的数据量 | ≤ 5 分钟 |
| RTO (Recovery Time Objective) | 恢复服务所需时间 | ≤ 30 分钟 |

## 二、架构设计

### 2.1 同城双活架构图

```
                          ┌──────────────────────────────┐
                          │     Global Traffic Manager    │
                          │   (GTM / CloudFront / ALB)   │
                          └──────────┬───────────────────┘
                              DNS / GeoRouting
                        ┌────────────┴────────────┐
                        │                         │
            ┌───────────▼───────────┐   ┌────────▼────────────┐
            │     Zone A (杭州)      │   │    Zone B (上海)     │
            │                       │   │                     │
            │  ┌─────────────────┐  │   │ ┌─────────────────┐ │
            │  │ Primary RDS     │  │   ││ Replica RDS      │ │
            │  │ MySQL 8.0       │◄─┼──►││ MySQL 8.0        │ │
            │  │ InnoDB Cluster  │  │   ││ Semi-sync Rep    │ │
            │  └─────────────────┘  │   │└─────────────────┘ │
            │                       │   │                     │
            │  ┌─────────────────┐  │   │ ┌─────────────────┐ │
            │  │ PolarDB读集群    │  │   ││ PolarDB只读节点   │ │
            │  │ 4x x86.large    │  │   ││ 2x x86.large     │ │
            │  └─────────────────┘  │   │└─────────────────┘ │
            │                       │   │                     │
            │  ┌─────────────────┐  │   │ ┌─────────────────┐ │
            │  │ Redis Cluster    │  │   ││ Redis Global     │ │
            │  │ 3主3从           │──┼──►││ Redis            │ │
            │  └─────────────────┘  │   ││ GlobalReplication│ │
            │                       │   │└─────────────────┘ │
            └───────────────────────┘   └─────────────────────┘
                      ↑ DataSync(DTS) ↑ Cross-region replication
```

### 2.2 技术选型对比

| 维度 | 阿里云 RDS | AWS RDS Multi-AZ | 华为云 GaussDB | 腾讯云 TDSQL | 天翼云 |
|------|-----------|-------------------|---------------|-------------|--------|
| 复制方式 | 半同步/异步 | InnoDB Replication | Redo日志复制 | Paxos协议 | - |
| 最大RPO | 0 (半同步) | ~秒级 | < 1s | 0 | - |
| 自动故障切换 | ✅ ≤30s | ✅ ≤60s | ✅ ≤30s | ✅ ≤30s | - |
| 读写分离 | PolarDB原生 | Read Replicas | DWS分离 | TDSQL Proxy | - |
| 跨AZ容灾 | 多可用区 | Multi-AZ | 三副本分布式 | 同城三单元 | - |

## 三、核心部署方案

### 3.1 MySQL 高可用集群 (PolarDB-X / RDS)

```bash
# ========== 阿里云 RDS 高可用版创建 ==========
# 使用 aliyun CLI 或 Terraform

# 方法一: Terraform (阿里云 provider)
# providers/alicloud/rds.tf
resource "alicloud_db_instance" "primary" {
  engine         = "MySQL"
  engine_version = "8.0"
  instance_type  = "rds.mysql.c8.xlarge"  # 8核16G
  instance_storage = 200                    # 200GB SSD
  
  # 关键配置: HA模式
  db_instance_class      = "mysql.c1.xlarge.sto3h"
  zone_id                = "cn-hangzhou-i"   # Zone A
  
  vswitch_id             = alicloud_vswitch.rds-vswitch.id
  security_group_ids     = [alicloud_security_group.rds-sg.id]
  
  # 高可用配置
  maintenance_time       = "02:00:00-03:00:00"
  backup_retention_period = 7
  backup_time_window     = "02:00:00-03:00:00"
  
  # 加密与审计
  kms_key_id             = alicloud_kms_key.rds-key.key_id
  parameter_group_id     = alicloud_db_parameter_group.mysql-strict.id
  
  tags = {
    Environment = "production"
    DR          = "multi-zone"
  }
}

resource "alicloud_db_instance" "secondary" {
  depends_on = [alicloud_db_instance.primary]
  
  engine         = "MySQL"
  engine_version = "8.0"
  instance_type  = "rds.mysql.c8.xlarge"
  instance_storage = 200
  
  # 关键: 不同可用区实现容灾
  zone_id        = "cn-hangzhou-j"   # Zone B
  
  master_instance_id = alicloud_db_instance.primary.id
  
  tags = {
    Role  = "read-replica"
    DR    = "multi-zone"
  }
}
```

```yaml
# 方法二: K8s + Operator 方案 (Vitess/Pgpool-II)
# k8s/mysql-cluster.yaml
apiVersion: databasevitess.io/v1alpha1
kind: VitessCluster
metadata:
  name: product-database
  namespace: database-prod
spec:
  global:
    imagePullPolicy: IfNotPresent
    vitessImage: vitess/lite:v18.0.0
  cells:
  - id: production
    servers:
    - cell: production-zone-a
      hostname: vitess-etcd-zone-a
    - cell: production-zone-b
      hostname: vitess-etcd-zone-b
  tablesplits:
    enabled: false
  shardReplicationNodesPerCell: 2
  topologyRealmAntiAffinity: true
  vtgate:
    minReplicas: 3
    maxReplicas: 6
    extraFlags:
      grpc_server_max_send_size: 100MB
  vtctld:
    enabled: true
  topology:
    cellLocalMode: per_cell_locking
    cellToCellRPCTimeoutSeconds: 5
```

### 3.2 PostgreSQL 流式复制 + Patroni

```yaml
# postgres-patroni-ha.yaml
# Kubernetes 上部署 PostgreSQL 高可用集群 (Patroni + etcd)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-patroni
  namespace: database-prod
spec:
  serviceName: postgres-ha
  replicas: 3
  selector:
    matchLabels:
      app: postgres-patroni
  template:
    metadata:
      labels:
        app: postgres-patroni
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgres-patroni
              topologyKey: topology.kubernetes.io/zone  # 跨可用区分布
      containers:
      - name: patroni
        image: docker.io/zalando/postgres-operator:1.12
        env:
        - name: PATRONI_POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: ETCD_HOSTS
          value: "etcd-0.etcd.database-prod.svc:2379,etcd-1.etcd.database-prod.svc:2379"
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
      - name: postgresql
        image: docker.io/postgres:16-alpine
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "1000m"
            memory: "1Gi"
          limits:
            cpu: "4000m"
            memory: "4Gi"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data-pvc
---
# Patroni 配置文件
apiVersion: v1
kind: ConfigMap
metadata:
  name: patroni-config
  namespace: database-prod
data:
  patroni.yaml: |
    postgresql:
      use_pg_rewind: true
      remove_data_directory_on_recover_false: true
      parameters:
        wal_level: replica
        max_wal_senders: 5
        max_replication_slots: 5
        hot_standby: "on"
        wal_keep_size: 1024
        synchronous_commit: on
        synchronous_standby_names: '*'
        checkpoint_timeout: 30min
        max_connections: 500
      retry_timeout: 10
      connect_address: $(POD_IP):5432
    etcd:
      hosts: "etcd-0.etcd.database-prod.svc:2379,etcd-1.etcd.database-prod.svc:2379,etcd-2.etcd.database-prod.svc:2379"
    scope: postgres-primary
    name: postgres-node
```

### 3.3 Redis 全球多活

```yaml
# redis-global-active.yaml
# 使用阿里云 Redis 全局数据库
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
  namespace: cache-prod
data:
  redis.conf: |
    # ==================== 基本配置 ====================
    port 6379
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    appendfsync everysec
    
    # ==================== 安全加固 ====================
    requirepass ${REDIS_PASSWORD}
    masterauth ${REDIS_PASSWORD}
    
    # ==================== 内存策略 ====================
    maxmemory-policy allkeys-lru
    maxmemory 4gb
    
    # ==================== 持久化 ====================
    save 900 1
    save 300 10
    save 60 10000
    
    # ==================== 慢查询日志 ====================
    slowlog-log-slower-than 10000
    slowlog-max-len 128
```

```bash
#!/bin/bash
# redis-health-check.sh - Redis 健康检查脚本
set -euo pipefail

REDIS_CLI="redis-cli -h ${REDIS_HOST:-127.0.0.1} -p ${REDIS_PORT:-6379} -a ${REDIS_PASSWORD}"

check_redis_health() {
    local health_info=$($REDIS_CLI INFO replication 2>/dev/null)
    
    # 获取角色信息
    ROLE=$(echo "$health_info" | grep "role:" | cut -d: -f2 | tr -d '[:space:]')
    CONNECTED_SLAVES=$(echo "$health_info" | grep "connected_slaves:" | cut -d: -f2 | tr -d '[:space:]')
    
    echo "[INFO] Redis Role: $ROLE"
    echo "[INFO] Connected Slaves: $CONNECTED_SLAVES"
    
    # 检查延迟
    if [ "$ROLE" == "master" ]; then
        # 统计每个 slave 的延迟
        for i in $(seq 0 $((CONNECTED_SLAVES - 1))); do
            SLOWDOWN=$($REDIS_CLI INFO replication 2>/dev/null | grep "slave${i}_lag_seconds" | cut -d: -f2 | tr -d '[:space:]')
            if [ ! -z "$SLOWDOWN" ] && [ "$SLOWDOWN" != "" ]; then
                echo "[SLAVE-$i] Lag: ${SLOWDOWN}s"
            fi
        done
        
        # 检查内存使用率
        USED_MEM_PCT=$($REDIS_CLI INFO memory 2>/dev/null | grep "used_memory_percentage" | cut -d: -f2 | tr -d '[:space:]')
        if [ ! -z "$USED_MEM_PCT" ]; then
            echo "[MEM] Used: ${USED_MEM_PCT}%"
            if (( $(echo "$USED_MEM_PCT > 90" | bc -l) )); then
                echo "[CRITICAL] Memory usage > 90%! Consider scaling up."
                exit 2
            fi
        fi
    fi
    
    # 检查客户端连接数
    CONN_COUNT=$($REDIS_CLI CLIENT LIST 2>/dev/null | wc -l)
    MAX_CONN=$($REDIS_CLI CONFIG GET maxclients 2>/dev/null | tail -1)
    echo "[CONN] Active: $CONN_COUNT / Max: $MAX_CONN"
    
    return 0
}

check_redis_health
```

### 3.4 DTS 数据迁移与同步

```bash
#!/bin/bash
# dts-sync-monitor.sh - DTS 同步延迟监控
# 阿里云 DTS Console API 查询同步状态
set -euo pipefail

ALIBABA_CLOUD_REGION="cn-shanghai"
ALIBABA_CLOUD_ACCESS_KEY_ID="${ALIYUN_AK:?AK not set}"
ALIBABA_CLOUD_ACCESS_KEY_SECRET="${ALIYUN_SK:?SK not set}"
DTS_JOB_ID="${DTS_TASK_ID:?DTS Job ID not set}"

query_sync_status() {
    local TIMESTAMP=$(date +%Y-%m-%dT%H:%M:%SZ)
    local DATE=$(date +%Y%m%dT%H%M%SZ)
    
    # 构造请求签名
    local string_to_sign="GET&%2F&AccessKeyId=${ALIBABA_CLOUD_ACCESS_KEY_ID}&Action=DescribeDtsJobs&Format=json&JobId=${DTS_JOB_ID}&SignatureMethod=HMAC-SHA1&SignatureNonce=${RANDOM}&SignatureVersion=1.0&Timestamp=${TIMESTAMP}&Version=2020-01-01"
    
    echo "Querying DTS job status..."
    curl -s "https://dts.aliyuncs.com/?Action=DescribeDtsJobs&JobId=${DTS_JOB_ID}&Format=json" \
      -H "Authorization: OSS ${ALIBABA_CLOUD_ACCESS_KEY_ID}:${SIGNATURE}" \
      | jq '.Jobs.DtsJob[] | {JobId, Status, SyncDirection, ErrorMsg}'
}

check_sync_lag() {
    # 通过 SQL 对比源库和目标库的行数差异
    SOURCE_CONN="mysql -h ${SOURCE_HOST} -u ${SOURCE_USER} -p${SOURCE_PASS} ${SOURCE_DB}"
    TARGET_CONN="mysql -h ${TARGET_HOST} -u ${TARGET_USER} -p${TARGET_PASS} ${TARGET_DB}"
    
    for TABLE in orders payments users; do
        SRC_COUNT=$($SOURCE_CONN -e "SELECT COUNT(*) FROM $TABLE;" -sN)
        TGT_COUNT=$($TARGET_CONN -e "SELECT COUNT(*) FROM $TABLE;" -sN)
        
        LAG=$((SRC_COUNT - TGT_COUNT))
        echo "[TABLE:$TABLE] Source: $SRC_COUNT  Target: $TGT_COUNT  Lag: $LAG rows"
        
        if [ "$LAG" -gt 10000 ]; then
            echo "[WARNING] Sync lag exceeds threshold for $TABLE"
        fi
    done
}

case "${1:-status}" in
    status) query_sync_status ;;
    lag) check_sync_lag ;;
    *) echo "Usage: $0 {status|lag}"; exit 1 ;;
esac
```

## 四、故障切换流程

### 4.1 自动故障检测

```yaml
# mysql-failover-check.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-failover-controller
  namespace: database-prod
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: failover-monitor
        image: custom/db-failover:latest
        command: ["/usr/local/bin/failover-monitor"]
        env:
        - name: PRIMARY_ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: primary-endpoint
        - name: SECONDARY_ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: secondary-endpoint
        - name: HEALTH_CHECK_INTERVAL
          value: "5"
        - name: FAIL_THRESHOLD
          value: "3"
        - name: ALERT_WEBHOOK
          value: "https://hooks.slack.com/services/xxx"
```

### 4.2 手动故障切换

```bash
#!/bin/bash
# manual-failover.sh - 数据库故障切换脚本
# ⚠️ 生产环境需人工确认执行

PRIMARY_RDS_ID="${PRIMARY_RDS_ID:?Primary RDS ID required}"
SECONDARY_RDS_ID="${SECONDARY_RDS_ID:?Secondary RDS ID required}"

echo "=========================================="
echo "  数据库故障切换操作"
echo "=========================================="
echo "主库:  $PRIMARY_RDS_ID"
echo "备库:  $SECONDARY_RDS_ID"
echo ""
read -p "确认执行故障切换？(yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "取消操作"
    exit 1
fi

# 步骤 1: 验证当前状态
echo "[1/6] 检查主库状态..."
aliyun rds DescribeDBInstanceStatus --DBInstanceId $PRIMARY_RDS_ID --region cn-hangzhou

# 步骤 2: 强制主备切换
echo "[2/6] 执行主备切换..."
SWITCH_RESULT=$(aliyun rds SwapRdnsReadWrite \
    --DBInstanceId $SECONDARY_RDS_ID \
    --region cn-shanghai)

echo "切换结果: $SWITCH_RESULT"

# 步骤 3: 等待新主库就绪
echo "[3/6] 等待新主库就绪..."
sleep 30
NEW_PRIMARY=$(echo "$SWITCH_RESULT" | jq -r '.DBInstanceId' )
echo "新主库ID: $NEW_PRIMARY"

# 步骤 4: 更新应用连接
echo "[4/6] 更新 DNS/CNAME..."
# 如果是阿里云 RDS:
aliyun dns UpdateDomainGroupRecord --RecordId "${DNS_RECORD_ID}" --Value "${NEW_PRIMARY}.rm-master.rds.aliyuncs.com"

# 步骤 5: 验证数据一致性
echo "[5/6] 数据一致性校验..."
./verify-data-consistency.sh --source $NEW_PRIMARY --target $OLD_SECONDARY

# 步骤 6: 通知团队
echo "[6/6] 发送告警通知..."
curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\":\"🔄 DB Failover completed! New primary: $NEW_PRIMARY at $(date)\"}"

echo "✅ 故障切换完成!"
```

## 五、成本估算

| 资源 | 规格 | 月费用估算 |
|------|------|-----------|
| RDS 高可用主实例 | 8C16G 200GB | ¥1,800/月 |
| RDS 高可用备实例 | 8C16G 200GB | ¥1,800/月 |
| PolarDB 读节点 | 4x 2C4G | ¥800/月 |
| Redis 集群版 | 2C8G×3 | ¥600/月 |
| DTS 数据同步 | 标准版 | ¥500/月 |
| GTM全局流量管理 | 企业版 | ¥200/月 |
| **合计** | | **¥5,700/月** |

## 六、面试考点

1. **RPO vs RTO**: RPO关注数据丢失量(时间)，RTO关注停机时间长短
2. **半同步复制**: 主库提交前至少一个备库收到并写入 WAL，RPO≈0但影响性能
3. **脑裂防护**: quorum机制保证只有一个主库接受写请求
4. **数据一致性**: binlog position校验 + 行计数校验 + MD5 hash校验三层保障
5. **渐进式切换**: DNS TTL控制 → 读写分离代理路由调整 → 完全切换三步走

## 七、课后练习

1. 用 Terraform 在阿里云搭建跨区域 RDS 高可用架构
2. 编写 Postgres Patroni 自动故障切换测试脚本
3. 使用 mysqldump + mysqlbinlog 模拟完整的数据恢复演练
4. 对比 Redis Sentinel 与 Redis Cluster 的高可用方案差异
