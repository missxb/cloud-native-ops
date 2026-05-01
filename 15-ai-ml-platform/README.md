# 项目15：企业级 AI/ML 平台架构

> **云平台**: AWS + 阿里云 | **难度**: ⭐⭐⭐⭐⭐ | **工具**: SageMaker, PAI, GPU 集群, Kubeflow, MLOps  
> **场景**: 电商推荐系统 + 大模型微调平台，需管理数据→训练→部署→监控全流程

---

## 📖 项目概述

某电商需要用户推荐引擎 + 搜索算法迭代，同时探索大模型应用。构建统一 ML 平台，包含特征平台、模型训练(GPU)、推理服务、MLOps 流水线。

### 学习目标
- ✅ SageMaker + PAI 托管训练/推理
- ✅ GPU 集群 (ECS GN7i / EC2 P4d) K8s 调度
- ✅ MLOps: 模型版本 → CI/CD → A/B 测试 → 监控
- ✅ 特征平台 (Feature Store) 特征复用
- ✅ LLM 推理 (vLLM/TGI) + 向量数据库

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      企业 AI/ML 平台架构                               │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────┐      │
│  │                    数据层                                   │      │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │      │
│  │  │ 数据湖   │ │ 实时日志 │ │ 用户画像 │ │ 标注数据 │     │      │
│  │  │ S3/OSS   │ │ Kafka    │ │ Feature  │ │ Label    │     │      │
│  │  │          │ │          │ │ Store    │ │ Studio   │     │      │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘     │      │
│  └───────┼─────────────┼─────────────┼─────────────┼────────┘      │
│          │             │             │             │                │
│  ┌───────┴─────────────┴──────┬──────┴─────────────┴────────┐      │
│  │                      特征工程 (Feature Engineering)       │      │
│  │  ┌──────────────────────────────────────────────────┐   │      │
│  │  │  离线特征: Spark/Hive/MaxCompute (全量处理)       │   │      │
│  │  │  实时特征: Flink (增量更新)                       │   │      │
│  │  │  Feature Store: 统一管理 → 训练/推理复用          │   │      │
│  │  └──────────────────────┬───────────────────────────┘   │      │
│  └─────────────────────────┼──────────────────────────────┘      │
│                            │                                       │
│  ┌─────────────────────────┴──────────────────────────────┐      │
│  │                    模型训练层                            │      │
│  │                                                          │      │
│  │  ┌─────────────────────┐  ┌─────────────────────┐       │      │
│  │  │ AWS SageMaker       │  │ 阿里云 PAI           │       │      │
│  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │       │      │
│  │  │ │ Training Jobs   │ │  │ │ DLC (容器训练)  │ │       │      │
│  │  │ │ (p4d.24xlarge)  │ │  │ │ (A100 GPU)     │ │       │      │
│  │  │ └─────────────────┘ │  │ └─────────────────┘ │       │      │
│  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │       │      │
│  │  │ │ Hyperparameter  │ │  │ │ DSW (交互式)   │ │       │      │
│  │  │ │ Tuning          │ │  │ │ JupyterLab     │ │       │      │
│  │  │ └─────────────────┘ │  │ └─────────────────┘ │       │      │
│  │  └─────────────────────┘  └──────────┬──────────┘       │      │
│  └──────────────────────────────────────┼───────────────────┘      │
│                                         │                          │
│  ┌──────────────────────────────────────┼───────────────────┐      │
│  │                                 ▼                          │      │
│  │                    模型管理                                 │      │
│  │  ┌──────────────────────────────────────────────────┐     │      │
│  │  │  Model Registry: 版本 + 元数据 + 审批流          │     │      │
│  │  │  Model Store: S3/OSS (模型文件 + 配置)           │     │      │
│  │  └──────────────────────┬───────────────────────────┘     │      │
│  └─────────────────────────┼──────────────────────────────┘      │
│                            │                                       │
│  ┌─────────────────────────┴──────────────────────────────┐      │
│  │                    推理服务层                            │      │
│  │                                                          │      │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────┐  │      │
│  │  │ 在线推理        │ │ 批量推理       │ │ LLM 服务   │  │      │
│  │  │ KServe/PAI-EAS │ │ SageMaker   │ │ vLLM/TGI   │  │      │
│  │  │ <10ms延迟       │ │ Batch Trf    │ │ + KV Cache │  │      │
│  │  │ (推荐/搜索)     │ │ (天级预测)   │ │ (对话/总结) │  │      │
│  │  └────────────────┘ └────────────────┘ └────────────┘  │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              MLOps Pipeline (Kubeflow / MLflow)            │      │
│  │                                                             │      │
│  │  数据 → 特征 → 训练 → 评估 → 注册 → 部署 → A/B → 监控    │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: AWS SageMaker 训练

```python
# SageMaker Training Job: 推荐模型训练 (PyTorch)
import sagemaker
from sagemaker.pytorch import PyTorch

sagemaker_session = sagemaker.Session()
role = "arn:aws:iam::xxx:role/sagemaker-execution-role"

# 训练配置
estimator = PyTorch(
    entry_point="train.py",
    source_dir="./src",               # 训练代码
    role=role,
    framework_version="2.2.0",
    py_version="py310",
    instance_count=2,                  # 分布式训练 ×2
    instance_type="ml.p4d.24xlarge",  # 8×A100 80GB
    
    # Spot 训练可节约 60-70% 成本
    use_spot_instances=True,
    max_wait=86400,          # 24h 最大等待
    
    metric_definitions=[
        {"Name": "auc", "Regex": "auc=([0-9.]+)"},
        {"Name": "logloss", "Regex": "logloss=([0-9.]+)"},
    ],
    
    # 输出到 S3
    output_path="s3://ml-models/recommender/",
    
    # 超参数
    hyperparameters={
        "embedding_dim": 128,
        "num_layers": 3,
        "batch_size": 4096,
        "epochs": 10,
        "learning_rate": 0.001,
    },
    
    # 自动续传
    checkpoint_s3_uri="s3://ml-checkpoints/recommender/",
)

# 启动训练
estimator.fit({"train": "s3://feature-store/recommender/train/"})
```

### Step 2: 阿里云 PAI DLC (容器训练)

```yaml
# PAI-DLC 提交训练任务 (DeepSpeed 分布式)
apiVersion: pai.alibabacloud.com/v1
kind: TrainingJob
metadata:
  name: recommender-training-v3
spec:
  # 资源配置
  resources:
    worker:
      instanceType: ecs.gn7i-c32g1.8xlarge  # 1×A10 24GB
      replicas: 4  # 4机器分布式训练 (DeepSpeed ZeRO-3)
  
  # 镜像
  image: registry-vpc.cn-beijing.aliyuncs.com/ml/training:latest
  
  # 命令
  command: |
    deepspeed --num_gpus=4 \
      train_recommender.py \
      --model_name deepfm \
      --train_data oss://feature-store/train/ \
      --val_data oss://feature-store/val/ \
      --feature_config oss://feature-store/config.yaml \
      --output_dir oss://model-store/recommender/v3/ \
      --epochs 20 \
      --batch_size 4096 \
      --learning_rate 0.001 \
      --deepspeed_config ds_config.json
  
  # 环境变量
  env:
    - name: PAI_TRAINING_JOB_NAME
      valueFrom:
        fieldRef: metadata.name
    - name: WANDB_PROJECT
      value: "recommender"
    - name: WANDB_API_KEY
      valueFrom:
        secretKeyRef:
          name: wandb-secret
          key: api-key
```

### Step 3: Feature Store (特征平台)

```python
# AWS SageMaker Feature Store
from sagemaker.feature_store.feature_group import FeatureGroup

# 创建特征组 (类似"特征表")
feature_group = FeatureGroup(
    name="user-real-time-features",
    sagemaker_session=sagemaker_session,
)

feature_group.create(
    record_identifier_name="user_id",
    event_time_feature_name="event_time",
    feature_definitions=[
        {"featureName": "user_id", "featureType": "String"},
        {"featureName": "purchase_count_7d", "featureType": "Integral"},
        {"featureName": "purchase_amount_7d", "featureType": "Fractional"},
        {"featureName": "avg_session_duration", "featureType": "Fractional"},
        {"featureName": "category_preference", "featureType": "String"},
        {"featureName": "last_active_time", "featureType": "Integral"},
    ],
    online_store_config={"EnableOnlineStore": True},  # 实时查询
    offline_store_config={  # 离线分析
        "S3StorageConfig": {
            "S3Uri": "s3://feature-store/offline/"
        }
    },
)

# 写入特征 (Flink 实时 or Spark 批量)
# 实时特征写入 (Kafka + SageMaker Feature Store)
import boto3
featurestore_runtime = boto3.client('sagemaker-featurestore-runtime')

featurestore_runtime.put_record(
    FeatureGroupName="user-real-time-features",
    Record=[
        {"FeatureName": "user_id", "ValueAsString": "u12345"},
        {"FeatureName": "purchase_count_7d", "ValueAsString": "15"},
        {"FeatureName": "purchase_amount_7d", "ValueAsString": "2340.5"},
    ]
)

# 推理时读取特征 (推荐模型在线推理)
response = featurestore_runtime.get_record(
    FeatureGroupName="user-real-time-features",
    RecordIdentifierValueAsString="u12345",
)
user_features = response['Record']  # 低延迟读取 (<10ms)
```

### Step 4: LLM 推理服务 (vLLM)

```yaml
# vLLM 推理部署 (HuggingFace 模型 + GPU)
# K8s Deployment 示例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-qwen-72b
  namespace: ai-inference
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-qwen
  template:
    metadata:
      labels:
        app: llm-qwen
    spec:
      nodeSelector:
        nvidia.com/gpu.product: "NVIDIA-A100"  # 2×A100
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - "--model"
          - "Qwen/Qwen-72B"
          - "--tensor-parallel-size"
          - "2"                    # 2 GPU 张量并行
          - "--max-model-len"
          - "8192"                 # 上下文长度
          - "--gpu-memory-utilization"
          - "0.90"
          - "--enable-prefix-caching"
          - "true"                 # 前缀缓存 (提升 ChatGPT类场景效率)
          - "--port"
          - "8000"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 2
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token
              key: token
        volumeMounts:
        - name: model-cache
          mountPath: /root/.cache/huggingface
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache-pvc

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: llm-qwen-service
spec:
  selector:
    app: llm-qwen
  ports:
  - port: 8000
    targetPort: 8000
```

### Step 5: MLOps Pipeline (Kubeflow)

```python
# Kubeflow Pipeline: 端到端 ML 工作流
from kfp import dsl, compiler

@dsl.component
def fetch_data(input_uri: str) -> str:
    """拉取训练数据"""
    # 从 Feature Store / 数据湖加载
    return input_uri

@dsl.component
def train_model(data_uri: str, hyperparams: dict) -> str:
    """训练模型"""
    # 调用 SageMaker/PAI 提交训练任务
    model_uri = f"s3://models/recommender/{datetime.now()}"
    return model_uri

@dsl.component
def evaluate_model(model_uri: str, val_data: str) -> dict:
    """评估模型"""
    # 对比基线模型
    metrics = {"auc": 0.872, "latency_p99": 15}
    return metrics

@dsl.component
def validate_metrics(metrics: dict, threshold: float) -> bool:
    """准入校验"""
    if metrics["auc"] < threshold:
        raise ValueError(f"AUC {metrics['auc']} < {threshold}")
    return True

@dsl.component
def register_model(model_uri: str, metrics: dict) -> str:
    """注册到 Model Registry"""
    version = "v1.23"
    return version

@dsl.component
def deploy_model(model_uri: str, version: str, env: str):
    """部署到推理服务 (KServe / PAI-EAS)"""
    # K8s KServe InferenceService
    pass

@dsl.pipeline(name="recommender-pipeline")
def recommender_pipeline():
    # 1. 数据准备
    data_op = fetch_data(input_uri="s3://feature-store/train/")
    
    # 2. 训练
    train_op = train_model(
        data_uri=data_op.output,
        hyperparams={"embedding_dim": 128, "batch_size": 4096}
    )
    
    # 3. 评估
    eval_op = evaluate_model(
        model_uri=train_op.output,
        val_data="s3://feature-store/val/"
    )
    
    # 4. 准入校验 (AUC > 0.85)
    validate_op = validate_metrics(
        metrics=eval_op.output,
        threshold=0.85
    )
    
    # 5. 注册
    register_op = register_model(
        model_uri=train_op.output,
        metrics=eval_op.output
    ).after(validate_op)
    
    # 6. 部署 (Staging)
    deploy_op = deploy_model(
        model_uri=train_op.output,
        version=register_op.output,
        env="staging"
    )

# 编译 Pipeline
compiler.Compiler().compile(recommender_pipeline, "recommender-pipeline.yaml")
```

### Step 6: 向量数据库 + RAG

```python
# 阿里云 Elasticsearch 8.x (向量检索能力)
# RAG: 检索增强生成

from elasticsearch import Elasticsearch
from sentence_transformers import SentenceTransformer

es = Elasticsearch(
    "https://es-xxx.elasticsearch.aliyuncs.com",
    http_auth=("elastic", "password")
)
model = SentenceTransformer("BAAI/bge-large-zh-v1.5")  # 中文 Embedding

# 写入商品向量 (用于语义搜索)
def index_product(product: dict):
    text = f"{product['title']} {product['description']}"
    vector = model.encode(text).tolist()
    
    es.index(index="products", body={
        "title": product["title"],
        "description": product["description"],
        "category": product["category"],
        "embedding": vector  # 1024维向量
    })

# 语义搜索
def semantic_search(query: str, top_k: int = 10):
    query_vector = model.encode(query).tolist()
    
    result = es.search(index="products", body={
        "knn": {
            "field": "embedding",
            "query_vector": query_vector,
            "k": top_k,
            "num_candidates": 100
        }
    })
    
    return result["hits"]["hits"]

# RAG 流程: 检索 → 增强 → 生成
def rag_generate(user_query: str):
    # 1. 检索相关文档
    docs = semantic_search(user_query, top_k=5)
    context = "\n".join([d["_source"]["description"] for d in docs])
    
    # 2. 构建 Prompt
    prompt = f"""基于以下商品信息回答用户问题:
    {context}
    
    用户问题: {user_query}"""
    
    # 3. 调用 LLM 生成
    response = openai.ChatCompletion.create(
        model="qwen-72b",
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.choices[0].message.content
```

---

## 📊 GPU 型号对比

| GPU | 显存 | 适用场景 | 参考价格/时 |
|-----|------|----------|-------------|
| NVIDIA A10 | 24GB | 小模型/搜广推 | ~¥12 (阿里云) |
| NVIDIA A100 | 40/80GB | 大模型训练/推理 | ~¥32 (阿里云) |
| NVIDIA H100 | 80GB | 最新大模型 | ~$12 (AWS) |
| L40S | 48GB | 推理+图形 | ~¥18 |

---

## 💰 成本估算

| 服务 | 月费 |
|------|------|
| SageMaker Training (100·h p4d) | ~$3,200 |
| PAI-DLC (200·h A100) | ~¥6,400 |
| KServe/PAI-EAS 推理 (4×A10) | ~¥12,000 |
| vLLM LLM 服务 (2×A100, 24/7) | ~¥23,000 |
| S3/OSS 模型存储 (5TB) | ~$150 |
| Feature Store | ~$500 |
| ES 向量数据库 (8C32G×3) | ~¥5,400 |
| Kubeflow Pipeline | ¥0 (自建 on K8s) |
| **合计** | **~$3,850 + ~¥46,800** |

---

## 📝 面试常见问题

**Q1: MLOps 和 DevOps 有什么区别？**
> MLOps 多了数据版本管理+特征工程+模型实验跟踪+模型漂移监控。DevOps 只管代码，MLOps 还要管数据+模型。

**Q2: Feature Store 解决了什么问题？**
> 训练和推理使用同一套特征定义，避免 Training-Serving Skew(训练推理不一致)。统一管理特征血缘+时效。

**Q3: LLM 推理如何优化成本？**
> vLLM 连续批处理+PagedAttention前缀缓存+FP8量化+SpeedCaching。吞吐可提升 10-20 倍。

**Q4: GPU 集群如何做资源管理？**
> K8s GPU Operator + MIG (多实例GPU) 切分 + Volcano/PriorityClass 排队调度 + 抢占式实例批处理。
