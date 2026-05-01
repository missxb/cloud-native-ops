# 项目十九: 容器存储 CSI 插件多厂商适配

## 一、项目背景

Kubernetes 中持久化存储的核心问题：
- **应用数据生命周期 ≠ Pod 生命周期**: Pod 重启/扩缩容需要保留数据
- **存储供应商锁定**: 不同云平台提供不同的块存储/NFS/文件系统
- **动态供给**: 手动创建 PV/PVC 无法满足弹性伸缩需求
- **快照与备份**: 合规要求数据库定期快照，跨区复制

CSI (Container Storage Interface) 是 CNCF 标准，解耦 K8s 与底层存储实现。

## 二、架构设计

### 2.1 CSI 架构

```
┌─────────────────────────────────────────────────────┐
│                Kubernetes Control Plane              │
│                                                      │
│  ┌──────────┐    Provisioner    ┌─────────────┐     │
│  │ kubelet   │ ◄───────────────►│ CSI Driver  │     │
│  │           │                  │ Controller   │     │
│  │           │                  │   (Stateful) │     │
│  │           │  NodeAttach       └──────┬──────┘     │
│  └─────┬─────┘                         │            │
│        │ gRPC                            │ gRPC       │
│        ▼                                ▼            │
│  ┌─────────────┐               ┌─────────────────┐   │
│  │ CSI Node     │               │ CSI Attacher    │   │
│  │ Driver(Pod)  │               │ /Registrar      │   │
│  │ - mount/unmount│             │                 │   │
│  └─────────────┘               └─────────────────┘   │
│       ▲                                                     │
│       │ RPC calls                                     │
│  ┌────┴────┐                                         │
│  │ Storage  │                                        │
│  │ Provider │                                        │
│  │(Cloud API)│                                       │
└──────────────────────────────────────────────────────┘
```

### 2.2 多云 CSI 对比

| 特性 | Alibaba Cloud CSI | AWS EBS CSI | Ceph RBD CSI | Longhorn |
|------|------------------|-------------|--------------|----------|
| 类型 | 块存储 | 块存储 | 块存储 | 分布式块 |
| 最大卷大小 | 32TB | 16TB | - | 由集群容量决定 |
| 快照支持 | ✅ | ✅ | ⚠️ | ✅ |
| 克隆支持 | ✅ | ✅ | ❌ | ✅ |
| 扩容支持 | ✅ | ✅ | ❌ | ✅ |
| 多挂载 | ❌ | ❌ | ✅ | ❌ |

## 三、核心组件部署

### 3.1 Alibaba Cloud CSI Plugin 安装

```bash
#!/bin/bash
# install-alicloud-csi.sh - 阿里云 CSI 插件安装脚本
set -euo pipefail

NAMESPACE=kube-system

echo "=== 安装阿里云 CSI Plugin ==="

# 1. 安装 csi-plugin (所有节点上的 node driver)
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/alibaba-cloud-csi-driver/master/docs/plugin.yaml

# 等待 csi-plugin DaemonSet ready
echo "等待 csi-plugin DaemonSet 就绪..."
kubectl rollout status daemonset/alibabacloud-csi-controller -n $NAMESPACE --timeout=300s
kubectl rollout status daemonset/alibabacloud-csi-node -n $NAMESPACE --timeout=300s

# 2. 安装 snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/sig-storage-lib-external-provisioner/deploy/k8s-1.18/snapshotter/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/sig-storage-lib-external-provisioner/deploy/k8s-1.18/snapshotter/snapshot-controller-deployment.yaml

# 3. 验证安装
kubectl get pods -n kube-system | grep alibabacloud
# expected:
# alibabacloud-csi-controller-xxx   3/3 Running
# alibabacloud-csi-node-xxx         2/2 Running
# alibabacloud-csi-node-yyy         2/2 Running
```

```yaml
# storageclass-high-iops.yaml
# 高性能 SSD 存储类（适用于数据库）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-essd-pl0
provisioner: disk.csi.alibabacloud.com
parameters:
  type: essd_PL0           # ESSD PL0 级别
  encryption_key_id: ""     # 可选: KMS密钥ID
  performance_level: PL0
reclaimPolicy: Delete      # 删除PVC时同时删除磁盘
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定到调度节点
allowVolumeExpansion: true  # 允许在线扩容
mountOptions:
- discard                   # TRIM/Discard 支持
---
# PL1 级别更高性能
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-essd-pl1
provisioner: disk.csi.alibabacloud.com
parameters:
  type: essd_PL1
  performance_level: PL1
reclaimPolicy: Retain     # 保留磁盘(需手动清理)
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# NFS 共享存储（适用于文件服务）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-nfs
provisioner: nfs.csi.alibabacloud.com
parameters:
  server: s-NFS.cn-hangzhou.nas.aliyuncs.com
  vers: "3"
  dir: /k8s-data
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

### 3.2 Dynamic PVC 申请

```yaml
# postgres-pvc-example.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: database-prod
spec:
  accessModes:
    - ReadWriteOnce     # RWO: 单个节点读写
  storageClassName: alicloud-disk-essd-pl0
  resources:
    requests:
      storage: 50Gi
  volumeMode: Filesystem  # Filesystem vs Block
---
# 读写多次存储（NFS/对象存储网关）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-documents
  namespace: app-prod
spec:
  accessModes:
    - ReadWriteMany       # RWX: 多个节点同时读写
  storageClassName: alicloud-nfs
  resources:
    requests:
      storage: 100Gi
---
# 预分配静态PV场景
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-db-volume
  labels:
    topology: zone-a
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: disk.csi.alibabacloud.com
    volumeHandle: d-bp1abc123def456    # 阿里云磁盘ID
    volumeAttributes:
      diskName: prod-db-disk
      encrypted: "true"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - cn-hangzhou-i
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-static-pvc
  namespace: database-prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 200Gi
  volumeName: static-db-volume    # 直接绑定已存在的PV
```

### 3.3 Snapshot & VolumeClone

```yaml
# volume-snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: alicloud-essd-snap-class
driver: disk.csi.alibabacloud.com
deletionPolicy: Delete          # 删除快照也会删除对应的快照对象
parameters:
  category: cloud_essd           # 快照来源的磁盘类型
  description: "Automated backup snapshot"
---
# 触发快照
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-pre-backup
  namespace: database-prod
spec:
  volumeSnapshotClassName: alicloud-essd-snap-class
  source:
    persistentVolumeClaimName: postgres-data
---
# 从快照恢复创建新PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored-from-snap
  namespace: database-prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: alicloud-disk-essd-pl0
  resources:
    requests:
      storage: 50Gi    # 必须 >= 源PVC大小
  dataSource:
    kind: VolumeSnapshot
    name: postgres-pre-backup
    apiGroup: snapshot.storage.k8s.io
```

### 3.4 Storage Capacity Tracker (自动发现可用容量)

```yaml
# storage-capacity-tracker-config.yaml
# SCS 让调度器了解各 Zone 的磁盘容量余量
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alibabacloud-scs-tracker
  namespace: kube-system
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: tracker
        image: registry.cn-hangzhou.aliyuncs.com/acs/csi-plugin:v1.34.2
        args:
        - --feature-gates=StorageCapacityTracker=true
        - --node-drain-timeout=2m
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

## 四、运维管理

### 4.1 存储日常巡检

```bash
#!/bin/bash
# storage-health-check.sh
set -euo pipefail

echo "=========================================="
echo "  Kubernetes CSI 存储健康检查"
echo "=========================================="

# 1. CSI Pods 状态
echo ""
echo "--- CSI Pods Status ---"
kubectl get pods -n kube-system -l app=(alibabacloud-csi-controller,alibabacloud-csi-node) \
  -o wide

# 2. Pending PVCs (可能有存储配额不足等问题)
echo ""
echo "--- Pending PVCs ---"
kubectl get pvc -A --field-selector=status.phase=Pending

# 3. StorageClass 用量统计
echo ""
echo "--- StorageClass Usage ---"
for sc in $(kubectl get sc -o name); do
    COUNT=$(kubectl get pvc -A -o jsonpath="{range .items[?(@.spec.storageClassName=='$(echo $sc | cut -d/ -f2)')]}{1}{end}" 2>/dev/null || echo "0")
    echo "  $sc: $COUNT PVCs"
done

# 4. CSI 事件日志
echo ""
echo "--- Recent CSI Events ---"
kubectl get events -n kube-system --field-selector reason=FailedMount --sort-by=.lastTimestamp | head -20

# 5. 磁盘使用率 (通过 CSI Node 端口)
echo ""
echo "--- CSI Node Export Metrics ---"
for pod in $(kubectl get pods -n kube-system -l component=csi-node -o name); do
    SVC_IP=$(kubectl get pod $pod -n kube-system -o jsonpath='{.status.podIP}')
    echo "  $pod ($SVC_IP)"
done

# 6. Volume 列表
echo ""
echo "--- Attached Volumes ---"
kubectl get pv -o custom-columns=\
  NAME:.metadata.name,\
  CAPACITY:.spec.capacity.storage,\
  CLASS:.storageClassName,\
  STATUS:.status.phase,\
  NODE:.spec.nodeAffinity.value 2>/dev/null | head -30
```

### 4.2 故障排查指南

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| PVC Pending | 存储配额耗尽 | `kubectl describe pvc` 查看事件；联系云厂商扩容 |
| 挂载超时 | EFS/CDFS 网络问题 | 检查 VPC 安全组规则是否放行 |
| Disk 不释放 | ReclaimPolicy=Retain | 手动删除对应 PV + 云盘资源 |
| CSI Controller CrashLoop | 认证失败 | 检查 SA token; 确认 RAM Role 配置 |
| I/O 性能下降 | ESSD 达到 IOPS 上限 | 升级磁盘等级或分拆为多个小盘 |

## 五、成本优化

```bash
# 识别未使用的卷
echo "=== 空闲 Volume 分析 ==="
kubectl get pv -o json | jq -r '
  [.[] | select(.spec.claimRef == null) | 
   {name: .metadata.name, size: .spec.capacity.storage, storageClass: .storageClassName}]
'

# 清理策略
# Retain Policy: 定期用脚本检测超过30天无绑定的PV并告警
while IFS= read -r pv; do
    AGE_DAYS=$(( ($(date +%s) - $(date -d "$(echo $pv | awk '{print $NF}')" +%s)) / 86400 ))
    if [ $AGE_DAYS -gt 30 ]; then
        echo "⚠️ PV $pv 已空闲 ${AGE_DAYS} 天，建议清理"
    fi
done < <(kubectl get pv --field-selector=status.phase=Available -o custom-columns="NAME:.metadata.name,AGE:.metadata.creationTimestamp")
```

### 成本估算

| 资源 | 规格 | 月费用 |
|------|------|--------|
| ESSD PL0 磁盘 | 每GB ¥0.33/月 | 按需 |
| ESSD PL1 磁盘 | 每GB ¥0.71/月 | 按需 |
| NAS 文件系统 | 每GB ¥0.29/月 | 按需 |
| 快照存储 | 每GB ¥0.05/月 | 按需 |
| CSI Controller | 2x2C4G | 自付 |

## 六、面试考点

1. **PVC vs PV**: PVC 是用户请求，PV 是物理存储抽象，两者通过 Binding 关联
2. **Provisioner 模式**: In-tree (内嵌) vs Out-of-tree (CSI独立进程)，后者更灵活可替换
3. **volumeBindingMode**: Immediate(立即绑定) vs WaitForFirstConsumer(延迟绑定，感知节点拓扑)
4. **accessMode**: RWO/RWX/ROX 三种访问模式的语义和使用场景
5. **ReclaimPolicy**: Delete(删PVC联动删PV), Retain(保留数据手动处理), Recycle(EFAPK废弃策略)

## 七、课后练习

1. 在 K8s 集群中部署阿里云 CSI 插件并完成 PVC/PV 绑定测试
2. 编写 StorageClass 实现按 Tier 分级存储 (PL0/PL1/NFS)
3. 使用 VolumeSnapshot 实现定时备份并验证快照恢复流程
4. 使用 Static PV 映射已有的云盘到特定可用性区
