# 项目二十四: 混合云 GPU 算力调度池 (K8s + Volcano + NVIDIA)

## 一、项目背景

AI/ML 训练推理面临的关键挑战：
- **GPU 碎片化**: 不同训练作业需求不同 GPU 数量/型号，难以有效利用
- **任务依赖**: DLRM 训练 → Feature Extraction → Model Serving 存在 DAG 依赖
- **批量调度**: 小规模训练任务不应抢占大模型训练节点资源
- **跨云弹性**: 训练高峰期本地 GPU 不够用，需要自动扩展到云上

### 多云 GPU 实例对比

| 云平台 | GPU 类型 | 显存 | vCPU | 内存 | 单价/小时 |
|--------|----------|------|------|------|----------|
| 阿里云 ecs.gn7i | A10 40G | 40GB | 16 | 120GB | ¥45/h |
| 阿里云 ecs.gn7 | A100 80G | 80GB | 32 | 480GB | ¥120/h |
| AWS p4de | A100 80G | 80GB×8 | 96 | 1152GB | $32/h |
| AWS g5 | A10G 24G | 24GB | 8 | 32GB | $3/h |
| 华为云ecs.ps6 | Ascend 910B | 64GB | 32 | 240GB | ¥85/h |
| 腾讯云gpu-h10 | H20 96G | 96GB | 16 | 520GB | ¥65/h |
| 天翼云 | RTX 4090 | 24GB | 16 | 64GB | ¥12/h |

## 二、架构设计

### 2.1 GPU 算力池架构图

```
┌─────────────────────────────────────────────────────────┐
│                Global Scheduler / Karpenter              │
│          (跨集群/跨云 Pod 调度与弹性)                     │
└──────┬──────────────────────────────┬───────────────────┘
       │                              │
       ▼                              ▼
┌───────────────┐           ┌────────────────────┐
│ Local Cluster  │           │ Cloud Burst Pool    │
│ (Edge/DataCenter)│         │ (ECS/AWS/GCP/PUB)   │
│               │             │                    │
│  ┌───────────┐│   Spot     │  ┌──────────────┐ │
│  │Volcano     │◄──Sync────►│  │Spot/Burst    │ │
│  │Scheduler   │  Queue     │  │Cluster       │ │
│  └─────┬─────┘             │  └──────┬───────┘ │
│        │                   │         │          │
│  ┌─────▼─────┐            │  ┌──────▼───────┐  │
│  │ GPU Node  │            │  │ GPU Node     │  │
│  │ A100×8    │            │  │ A100×8       │  │
│  │ A10×4     │            │  │ A10×4        │  │
│  └───────────┘            │  └──────────────┘  │
└───────────────────────────┴────────────────────┘
```

### 2.2 技术选型

| 功能 | 开源方案 | 说明 |
|------|---------|------|
| GPU 插件 | NVIDIA Device Plugin | 自动发现和暴露 GPU 资源 |
| 批处理调度器 | Volcano | Kubernetes 原生 Job 编排引擎 |
| GPU 共享 | MIG (A100) / vGPU (T4) | 多租户隔离 |
| 动态伸缩 | Cluster Autoscaler / Karpenter | 按需添加 GPU 节点 |
| 镜像优化 | NVIDIA Container Toolkit | CUDA/cuDNN/runtime 打包 |

## 三、核心部署方案

### 3.1 NVIDIA GPU Operator 安装

```bash
#!/bin/bash
# install-gpu-operator.sh - NVIDIA GPU Operator 一键安装
set -euo pipefail

echo "=== Installing NVIDIA GPU Operator ==="

# 1. Add NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia \
  && helm repo update

# 2. Create namespace
kubectl create namespace gpu-operator

# 3. Install GPU Operator with default profile
helm install --wait --timeout=300s \
  gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --set driver.enabled=true \
  --set driver.version=535.104.12 \
  --set node-feature-discovery.enabled=true \
  --set dcp.enabled=true \
  --set migStrategy=mixed \
  --set containerRuntimeProvider=auto \
  --set toolkit.enabled=true \
  --set devicePlugin.enabled=true \
  --set mig.manager.enabled=true

# 4. Verify all components are running
kubectl get pods -n gpu-operator
# Expected:
# gpu-collector-xxxxx    1/1 Running
# gpu-feature-discovery-xxxxx   1/1 Running
# mig-manager-xxxxx      1/1 Running
# nvidia-container-toolkit-xxxxx 1/1 Running (DaemonSet)
# nvidia-dcgm-xxxxx      1/1 Running (DaemonSet)
# nvidia-device-plugin-xxxxx   1/1 Running (DaemonSet)
# gpu-operator-validator-xxxxx  1/1 Running

# 5. Verify GPU is visible in nodes
kubectl describe nodes | grep -A5 "NVIDIA-A100"
# Output should show nvidia.com/gpu: "8" allocatable resources
```

```yaml
# gpu-node-profile.yaml
# 使用 Node Feature Discovery 标记 GPU 节点
apiVersion: node.k8s.io/v1
kind: NFDWorker
metadata:
  name: nfd-worker
  namespace: gpu-operator
spec:
  extraLabelDirs:
  - /etc/kubernetes/node-feature-discovery/labels.d
  kubeletConfigDir: /var/lib/kubelet/config.yaml
  sleepInterval: 60s
---
# 定义 GPU 节点标签规则
apiVersion: v1
kind: ConfigMap
metadata:
  name: nfd-master-conf
  namespace: gpu-operator
data:
  core:
    extraLabelSources: ['pci', 'system', 'usb']
    labelWhiteList:
    - 'hostname'
    - 'kernel.name'
  work:
    kubeletConfPath: /var/lib/kubelet/config.yaml
    cpuLabelSource: cpuid
    sleepInterval: 60s
  sources:
    pci:
      deviceClassWhitelist:
      - "02*"    # Network
      - "03*"    # Display/GPU
      - "12*"    # PCIe
      deviceFields:
        vendors:
        - "10de"  # NVIDIA
---
# 等待 GPU 就绪信号
# nvidia.com/gpu.present=true
# kubernetes.io/gpu=available
```

### 3.2 Volcano 批处理调度器部署

```bash
#!/bin/bash
# install-volcano.sh - Volcano 安装脚本
set -euo pipefail

echo "=== Installing Volcano Scheduler ==="

# 1. 添加 Volcano Helm Chart
helm repo add volcano-sh https://volcano-sh.github.io/charts
helm repo update

# 2. 创建火山作业命名空间
kubectl create namespace volcano-system

# 3. 安装 Volcano
helm install volcano volcano-sh/volcano \
  -n volcano-system \
  --version 1.9.0 \
  --set controller.image.tag=v1.9.0 \
  --set scheduler.image.tag=v1.9.0 \
  --set admission.enabled=true \
  --set priorityclass.enabled=true \
  --set metrics.enable=true

# 4. 验证
kubectl get pods -n volcano-system -l app.kubernetes.io/component=volcano
kubectl get pod -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.schedulerName}{"\n"}{end}'
# 显示所有使用了 volcano 调度器的 pod
```

```yaml
# volcano-cluster-queue.yaml
# Volcano 队列和集群队列配置
apiVersion: batch.volcano.dev/v1alpha1
kind: Queue
metadata:
  name: ml-training
spec:
  weight: 100
  reclaimable: true
  capability: resource.Volumes resource.Quantity
  min:
    cpu: "32"
    memory: 128Gi
  max:
    cpu: "256"
    memory: 1Ti
  status:
    state: Open
---
# 集群队列（跨多个 Queue）
apiVersion: batch.volcano.dev/v1alpha1
kind: ClusterQueue
metadata:
  name: ml-cluster-queue
spec:
  queue: ml-training
  namespaceSelector:
    matchLabels:
      ml-workloads: "true"
  namespaceSelectorMatchCriteria: ["include"]
---
# Priority Class 定义
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-training
value: 1000000
globalDefault: false
description: "High priority for LLM training jobs"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority-inference
value: 500000
globalDefault: false
description: "Medium priority for model inference"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority-batch
value: 100000
globalDefault: false
description: "Low priority for data preprocessing batch jobs"
```

### 3.3 训练作业配置

```yaml
# training-job-volcano.yaml
# BERT pre-training job using Volcano Job
apiVersion: batch.volcano.dev/v1alpha1
kind: VCJob
metadata:
  name: bert-pretrain
  namespace: ml-workloads
  labels:
    job-name: bert-pretrain
    team: ml-platform
spec:
  # 队列和优先级
  queue: ml-training
  priorityClassName: high-priority-training
  
  # 最小资源分配
  minAvailable: 2
  
  # PVC 共享模式
  vcjobs:
  - name: trainer
    policy: Terminated
    replicas: 8           # 8 卡分布式训练
    template:
      metadata:
        annotations:
          volcano.sh/profile: "trainer"
      spec:
        schedulerName: volcano  # 关键!指定 Volcano 调度器
        # 反亲和性确保每个 Pod 在不同节点上
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: nvidia.com/gpu.product
                  operator: In
                  values:
                  - NVIDIA-A100-SXM4-80GB
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - bert-pretrain-trainer
              topologyKey: kubernetes.io/hostname
        
        containers:
        - name: trainer
          image: registry.example.com/ml/bert-pretrain:v1.0.0
          command: ["python", "-m", "torch.distributed.run", "--nproc_per_node=8", "train.py"]
          env:
          - name: NCCL_DEBUG
            value: INFO
          - name: NCCL_SOCKET_IFNAME
            value: eth0
          - name: MASTER_ADDR
            valueFrom:
              configMapKeyRef:
                name: nccl-config
                key: master_addr
          - name: MASTER_PORT
            value: "29500"
          - name: WORLD_SIZE
            value: "8"
          resources:
            requests:
              cpu: "32"
              memory: "500Gi"
              nvidia.com/gpu: "8"   # 请求 8 块 A100
            limits:
              cpu: "32"
              memory: "500Gi"
              nvidia.com/gpu: "8"
          volumeMounts:
          - name: model-checkpoint
            mountPath: /mnt/checkpoints
          - name: dataset
            mountPath: /mnt/dataset
          - name: nvidia-install
            mountPath: /usr/local/nvidia
        volumes:
        - name: model-checkpoint
          persistentVolumeClaim:
            claimName: bert-checkpoints-pvc
        - name: dataset
          persistentVolumeClaim:
            claimName: bert-dataset-pvc
        - name: nvidia-install
          hostPath:
            path: /usr/local/nvidia
            type: DirectoryOrCreate
      
  # SubJobs: 依赖顺序执行
  minReady: 1
  schedulePolicy:
    FIFO: true
```

```yaml
# inference-serving-job.yaml
# 基于 KServe/Seldon 的模型推理服务
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: bert-classifier
  namespace: inference-prod
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  predictor:
    serviceAccountName: kserve-sa
    maxReplicas: 10
    minReplicas: 2
    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
        nvidia.com/gpu: "1"
      limits:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: "1"
    hardware:
      gpuName: "A100"
      gpuMemoryMin: "40Gi"
    model:
      modelFormat:
        name: huggingface
        version: "4.x"
      storageUri: s3://ml-models/bert-finetuned/
      args:
      - --model_name_or_path
      - bert-base-chinese
      - --max_length
      - "512"
      - --batch_size
      - "32"
    autoscaleTargetConcurrency: 10
    scaleTargetPercent: 80
```

### 3.4 弹性伸缩 - GPU 节点自动扩容

```yaml
# cluster-autoscaler-gpu-config.yaml
# CA 针对 GPU 节点的专用配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-autoscaler
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --cloud-provider=alicloud
        - --nodes=3:20:ecs.gn7i-c8g1.xlarge    # min:max:instance-type
        - --nodes=1:10:ecs.gn7-i-a100-k80.26g # min:max:GPU instance type
        - --scale-down-utilization-threshold=0.3
        - --scan-interval=10s
        - --expander=random
        - --balance-similar-node-groups=true
        - --skip-nodes-with-local-storage=false
        - --skip-nodes-with-system-pods=false
        env:
        - name: ALIBABA_CLOUD_REGION_ID
          value: cn-hangzhou
        - name: GOOGLE_APPLICATION_CREDENTIALS_JSON
          valueFrom:
            secretKeyRef:
              name: ca-cloud-creds
              key: credentials.json
```

### 3.5 GPU 监控 - DCGM Exporter

```yaml
# dcgm-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dcgm-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      name: dcgm-exporter
  template:
    metadata:
      labels:
        name: dcgm-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9400"
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - name: dcgm-exporter
        image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.4-3.4.0-ubuntu22.04
        securityContext:
          runAsNonRoot: false
        ports:
        - containerPort: 9400
        volumeMounts:
        - name: certs
          mountPath: /etc/dcgm/mgmt-certs
          readOnly: true
        resources:
          limits:
            nvidia.com/gpu: 0  # DCGM itself doesn't need GPU
      volumes:
      - name: certs
        hostPath:
          path: /etc/dcgm/mgmt-certs
```

```sql
-- Prometheus GPU Metrics Queries

-- GPU Utilization (%)
rate(nv_dcgmi_gpu_util[5m])

-- GPU Memory Used (MB)
nv_dcgmi_mem_used

-- GPU Temperature (°C)
nv_dcgmi_temp_current

-- GPU Power Consumption (W)
nv_dcgmi_power_usage

-- GPU Error Count
nv_dcgmi_ecc_errors_uncorrected_total
```

## 四、运维管理

### 4.1 GPU 巡检清单

```bash
#!/bin/bash
# gpu-health-check.sh
set -euo pipefail

echo "=========================================="
echo "  GPU Health Check Report"
echo "=========================================="

for node in $(kubectl get nodes -l nvidia.com/gpu.present=true -o name); do
    echo ""
    echo "--- Node: $node ---"
    
    # SSH to node (or use kubectl exec on a GPU pod)
    kubectl exec -n gpu-operator \
      $(kubectl get pods -n gpu-operator -l component=nvidia-device-plugin -o name | head -1) \
      -- bash -c "dcgmi diag -r 0-7 2>/dev/null | grep -E 'Result|Status'"
    
    # Check for ECC errors
    kubectl top node "$node" 2>/dev/null || echo "(no metrics-server)"
done

# 查看 GPU 利用率 Top 10
echo ""
echo "--- Top 10 GPU Pods by Utilization ---"
kubectl get pods --sort-by="{.status.allocatable['nvidia\.com/gpu']}" -A | head -10
```

### 4.2 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Pod Pending 因 GPU | 集群 GPU 已用满 | 触发 CA 扩容; 或使用 Spot 实例 |
| NCCL 通信失败 | NVLink/PCIe 拓扑不优 | 设置 `NCCL_SOCKET_IFNAME` + 同一节点内通信 |
| OOM Killed | GPU 显存不足 | 增大 `gpuMemoryMin`; 减小 batch_size |
| MIG 分区不可见 | MIG manager 未启动 | 确认 `mig.manager.enabled=true` |
| RDMA 慢 | IB/RoCE 驱动未加载 | `ibstatus` 检查; 安装 `rdma-core` |

## 五、成本估算

| 资源 | 规格 | 月费用估算 |
|------|------|-----------|
| Volcano Scheduler | 免费开源 | 自付 K8s 资源 |
| 训练节点 (A100×8) | ecs.gn7-i-a100-k80.26g | ¥8,600/月 × 节点数 |
| 推理节点 (A10) | ecs.gn7i-c8g1.xlarge | ¥3,240/月 × 节点数 |
| NVMe SSD 系统盘 | 500GB ESSD | ¥165/月 |
| EIP (外部访问) | 按流量计费 | ¥~200/月 |

## 六、面试考点

1. **GPU 调度 vs CPU**: GPU 是独占资源不可超分(CPU可超分4-10x); Volcano 提供 binpacking
2. **NVLink vs PCIe**: NVLink 带宽 ~600GB/s vs PCIe 4.0 x16 ~32GB/s; 同节点内优先 NVLink
3. **MIG (Multi-Instance GPU)**: A100/H100 支持将单卡切分为 7 个独立实例，提升利用率
4. **分布式训练**: Data Parallel (数据并行) vs Model Parallel (模型并行) vs Pipeline Parallel (流水线)
5. **PyTorch DDP**: DistributedDataParallel 通过 NCCL backend 做 AllReduce 梯度同步

## 七、课后练习

1. 在 K8s 集群中部署 NVIDIA GPU Operator 并验证 A100 可见性
2. 编写 Volcano Job YAML 实现 4 卡 BERT 分布式预训练
3. 配置 Cluster Autoscaler 让 GPU 节点根据 pending pod 自动扩容
4. 使用 DCGM Exporter + Grafana 搭建 GPU 实时监控仪表盘
