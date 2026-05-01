# 项目06：Serverless 事件驱动架构

> **云平台**: AWS + 腾讯云 | **难度**: ⭐⭐⭐⭐ | **工具**: Lambda, SCF, EventBridge, Step Functions, API Gateway  
> **场景**: 图片/视频处理平台，完全 Serverless，按调用付费，零运维

---

## 📖 项目概述

某 UGC 平台用户上传图片/视频后需要缩略图、水印、审核、CDN 分发。日处理量 100 万+ 图片，峰值不可预测。采用全 Serverless 事件驱动架构。

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Serverless 事件驱动图片处理                          │
│                                                                      │
│  用户上传                                                            │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────┐    Event     ┌─────────────┐                           │
│  │  S3/COS  │────────────►│ EventBridge  │  ← 事件总线               │
│  │(对象存储)│  s3:PutObject│  (路由规则)   │                           │
│  └─────────┘              └──┬───┬───┬───┘                          │
│                              │   │   │                               │
│              ┌───────────────┘   │   └──────────────┐               │
│              ▼                   ▼                   ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │
│  │ Lambda/SCF  │    │ Lambda/SCF   │    │ Lambda/SCF   │           │
│  │ 图片压缩     │    │ 内容审核      │    │ AI 标签      │           │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘           │
│         │                   │                   │                    │
│         ▼                   ▼                   ▼                    │
│  ┌──────────┐      ┌──────────────┐    ┌──────────────┐            │
│  │结果存入S3 │      │审核不过→删除  │    │标签写入DB    │            │
│  └──────────┘      │或→人工审核队列│    │              │            │
│                    └──────────────┘    └──────────────┘            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │   API Gateway → Lambda/SCF (查询接口, 毫秒级响应)         │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │   Step Functions (AWS) / 腾讯云 ASW (工作流编排)          │       │
│  │   视频处理: 转码 → 切片 → 加密 → CDN推送 → 通知          │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: AWS Lambda (图片压缩)

```javascript
// AWS Lambda - 图片压缩 + 水印 (Node.js 18 + Sharp)
const AWS = require('aws-sdk');
const sharp = require('sharp');
const s3 = new AWS.S3();

exports.handler = async (event) => {
  // 从 EventBridge 事件中提取上传信息
  const bucket = event.detail.bucket.name;
  const key = decodeURIComponent(event.detail.object.key.replace(/\+/g, ' '));
  
  // 获取原图
  const original = await s3.getObject({ Bucket: bucket, Key: key }).promise();
  
  // 生成 3 种尺寸缩略图
  const sizes = [
    { suffix: 'thumb', width: 200 },
    { suffix: 'medium', width: 800 },
    { suffix: 'large', width: 1600 },
  ];
  
  const tasks = sizes.map(async ({ suffix, width }) => {
    const resized = await sharp(original.Body)
      .resize(width, null, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 85 })
      .toBuffer();
    
    const newKey = key.replace(/^original\//, `${suffix}/`);
    await s3.putObject({
      Bucket: process.env.OUTPUT_BUCKET,
      Key: newKey,
      Body: resized,
      ContentType: 'image/jpeg',
      Metadata: { compressed: 'true' },
    }).promise();
  });
  
  await Promise.all(tasks);
  
  // 返回结果给 EventBridge 下游
  return {
    statusCode: 200,
    body: JSON.stringify({ original: key, processed: true }),
  };
};

// Lambda 配置 (Terraform)
resource "aws_lambda_function" "image_resize" {
  function_name = "image-resize-processor"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "nodejs18.x"
  handler       = "index.handler"
  timeout       = 15
  memory_size   = 2048  # 2GB (Sharp 需要足够内存)
  
  # 从 S3 加载代码包
  s3_bucket = aws_s3_bucket.lambda_code.id
  s3_key    = "image-resize-v1.0.zip"
  
  environment {
    variables = {
      OUTPUT_BUCKET = aws_s3_bucket.processed.id
      LOG_LEVEL     = "info"
    }
  }
  
  tracing_config {
    mode = "Active"  # X-Ray 追踪
  }
}
```

### Step 2: EventBridge 事件路由

```hcl
# AWS EventBridge Rule: S3 上传 → Lambda 处理
resource "aws_cloudwatch_event_rule" "s3_upload" {
  name        = "s3-image-upload"
  description = "Trigger image processing on S3 upload"
  
  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail_type = ["Object Created"]
    detail = {
      bucket = { name = ["uploads-bucket"] }
      object = {
        key = [{ prefix = "original/" }]
        size = [{ numeric = [">=", 0] }]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.s3_upload.name
  target_id = "image-resize"
  arn       = aws_lambda_function.image_resize.arn
}
```

### Step 3: 腾讯云 SCF + 内容审核

```python
# 腾讯云 SCF - 内容审核 (Python)
import os
import json
from qcloud_cos import CosConfig, CosS3Client
from tencentcloud.common import credential
from tencentcloud.cms.v20190321 import cms_client, models

def main_handler(event, context):
    """COS 触发 → 内容审核 → 处理结果"""
    cos_event = event['Records'][0]['cos']
    bucket = cos_event['cosBucket']['name']
    key = cos_event['cosObject']['key']
    
    # 调用腾讯云内容安全审核
    cred = credential.Credential(
        os.environ['TENCENT_SECRET_ID'],
        os.environ['TENCENT_SECRET_KEY']
    )
    client = cms_client.CmsClient(cred, "ap-guangzhou")
    
    # 获取图片内容进行审核
    cos_client = CosS3Client(CosConfig(Region="ap-guangzhou"))
    img_content = cos_client.get_object(Bucket=bucket, Key=key)['Body'].read()
    
    req = models.ImageModerationRequest()
    req.FileContent = base64.b64encode(img_content).decode()
    
    resp = client.ImageModeration(req)
    
    if resp.Data.Suggestion == 'Block':
        # 违规 → 删除 + 通知
        cos_client.delete_object(Bucket=bucket, Key=key)
        send_alert(key, resp.Data.Label)
        return {"status": "blocked", "reason": resp.Data.Label}
    
    return {"status": "passed", "label": resp.Data.Label}
```

### Step 4: Step Functions 视频处理工作流

```json
{
  "Comment": "视频处理工作流 (AWS Step Functions)",
  "StartAt": "Transcode",
  "States": {
    "Transcode": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:xxx:function:video-transcode",
      "Next": "Choice"
    },
    "Choice": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.needsReview",
          "BooleanEquals": true,
          "Next": "ContentReview"
        }
      ],
      "Default": "GenerateThumbnails"
    },
    "ContentReview": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "content-review",
        "Payload": { "token.$": "$$.Task.Token", "input.$": "$" }
      },
      "TimeoutSeconds": 86400,
      "Catch": [
        {
          "ErrorEquals": ["States.Timeout"],
          "ResultPath": "$.error",
          "Next": "NotifyFailure"
        }
      ],
      "Next": "GenerateThumbnails"
    },
    "GenerateThumbnails": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "Thumbnail",
          "States": {
            "Thumbnail": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:xxx:thumbnails",
              "End": true
            }
          }
        },
        {
          "StartAt": "AI",
          "States": {
            "AI": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:xxx:ai-tagging",
              "End": true
            }
          }
        }
      ],
      "Next": "CDN Push"
    },
    "CDN Push": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:xxx:cdn-invalidate",
      "Next": "NotifySuccess"
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:xxx:video-events",
        "Message.$": "$",
        "Subject": "Video Processing Complete"
      },
      "End": true
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:xxx:video-events",
        "Message.$": "$.error",
        "Subject": "Video Processing Failed"
      },
      "End": true
    }
  }
}
```

---

## 💰 全 Serverless 成本 (100万图片/月)

| 计算 | AWS | 腾讯云 |
|------|-----|--------|
| Lambda/SCF 执行 | ~$5 | ~¥18 |
| S3/COS 存储 (1TB) | ~$23 | ~¥100 |
| API Gateway | ~$3.5 | ~¥15 |
| EventBridge | ~$1 | 免费 |
| Step Functions | ~$3 | ~¥10 |
| CDN 流量 | ~$50 | ~¥300 |
| **合计** | **~$85/月** | **~¥443/月** |

> 对比传统 ECS 方案: 至少 ¥3000/月 (2台常备服务器)

---

## 📝 面试常见问题

**Q1: Lambda 冷启动怎么解决？**
> Provisioned Concurrency (预置并发) 保持热实例，或设置 1 分钟的 CloudWatch Events 触发器保持 warm。

**Q2: 如何保证事件不丢失？**
> EventBridge 默认 at-least-once 投递 + DLQ (死信队列)。Lambda 异步调用有 2 次重试。

**Q3: 腾讯云和 AWS Serverless 生态差异？**
> AWS 更成熟 (Lambda/StepFunctions/EventBridge/AppSync)，腾讯云 SCF 起步晚但 COS 触发 + API 网关体系完整。
