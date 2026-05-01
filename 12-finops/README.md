# 项目12：FinOps 多云成本管理

> **云平台**: 阿里云 + AWS + 腾讯云 | **难度**: ⭐⭐⭐ | **工具**: 云成本分析, 预算管理, 资源优化, InfraCost  
> **场景**: 每月云支出超 ¥50 万，需要通过 FinOps 体系实现可观测、可优化、可预测

---

## 📖 项目概述

某企业 SaaS 使用阿里云+腾讯云+AWS，月支出 50 万+，资源利用率平均 30%。建立 FinOps 成本管理体系，目标 6 个月内降低 30% 云成本。

### 学习目标
- ✅ 多云成本可视化与分析
- ✅ 资源优化策略 (规格调整/预留实例/抢占式)
- ✅ 预算与告警自动化
- ✅ InfraCost + Terraform 成本预测
- ✅ FinOps 团队文化建设

---

## 🏗️ 成本管理体系

```
┌─────────────────────────────────────────────────────────────────────┐
│                       FinOps 成本管理平台                              │
│                                                                      │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │
│  │   阿里云       │  │    AWS        │  │   腾讯云       │           │
│  │ 费用中心/API  │  │ Cost Explorer │  │ 费用中心/API  │           │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘           │
│          │                  │                  │                     │
│  ┌───────┴──────────────────┴──────────────────┴───────┐            │
│  │              统一成本采集层 (Prometheus + Exporter)   │            │
│  └──────────────────────┬──────────────────────────────┘            │
│                         ▼                                            │
│  ┌───────────────────────────────────────────────────────┐          │
│  │                 成本分析平台                            │          │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │          │
│  │  │ 趋势分析     │ │ 部门分摊    │ │ 异常检测     │     │          │
│  │  │ (Grafana)   │ │ (Tag分账)   │ │ (告警)      │     │          │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │          │
│  └──────────────────────┬──────────────────────────────┘            │
│                         ▼                                            │
│  ┌───────────────────────────────────────────────────────┐          │
│  │                   优化执行层                            │          │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │          │
│  │  │右规格    │ │预留实例  │ │抢占实例  │ │闲置资源  │ │          │
│  │  │Rightsize │ │Reserved  │ │  Spot    │ │  回收    │ │          │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │          │
│  └───────────────────────────────────────────────────────┘          │
│                                                                      │
│  ┌───────────────────────────────────────────────────────┐          │
│  │              InfraCost: IaC 成本预览                    │          │
│  │       Terraform → Plan → 成本预估 → 审批 → Apply      │          │
│  └───────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: 成本可视化 (Grafana)

```yaml
# Prometheus Exporter: 采集阿里云账单 (自建或开源)
# 开源方案: cloudcost-exporter / aliyun-billing-exporter

apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudcost-exporter-config
  namespace: observability
data:
  config.yaml: |
    clouds:
      aliyun:
        access_key_id: ${ALIYUN_AK}
        access_key_secret: ${ALIYUN_SK}
        region: cn-hangzhou
        collect_interval: 3600  # 每小时采集一次
      aws:
        region: us-east-1
        # 使用 IRSA 自动获取凭证
        collect_interval: 3600
      tencent:
        secret_id: ${TENCENT_ID}
        secret_key: ${TENCENT_KEY}
        collect_interval: 3600
```

```sql
-- Grafana 成本分析 SQL (基于 Prometheus)
-- 按服务类型的日月成本
sum by (service) (
  avg_over_time(cloud_cost_daily_dollars{cloud=~"$cloud"}[$__range])
)

-- 按部门 (Resource Tag) 分摊
sum by (team) (
  cloud_cost_hourly_dollars * 730  -- 月化
)

-- 同比环比 (减少/增加百分比)
(
  sum(cloud_cost_monthly_dollars{month="current"}) 
  - sum(cloud_cost_monthly_dollars{month="previous"})
) / sum(cloud_cost_monthly_dollars{month="previous"}) * 100
```

### Step 2: 资源优化策略

```python
# 资源优化分析脚本 (Python)
# 1. Rightsizing: 识别过度配置的实例
def analyze_ecs_oversized():
    """分析 ECS/EC2 利用率"""
    oversized = []
    for instance in get_all_instances():
        cpu_avg = get_cpu_utilization(instance.id, days=30)
        mem_avg = get_memory_utilization(instance.id, days=30)
        
        # 30天平均 CPU < 20% 且 内存 < 40%
        if cpu_avg < 20 and mem_avg < 40:
            # 建议降级
            recommended = get_next_lower_spec(instance.type)
            savings = estimate_cost_savings(instance.type, recommended)
            
            oversized.append({
                'instance_id': instance.id,
                'current_type': instance.type,
                'cpu_avg': cpu_avg,
                'mem_avg': mem_avg,
                'recommended': recommended,
                'monthly_savings': savings,
            })
    
    return sorted(oversized, key=lambda x: x['monthly_savings'], reverse=True)

# 2. 预留实例推荐
def recommend_reserved_instances():
    """推荐购买预留实例"""
    # 分析过去 60 天持续运行的按量付费实例
    on_demand = get_on_demand_instances(running_days__gte=60)
    
    # 计算 ROI
    recommendations = []
    for instance in on_demand:
        monthly_ondemand = get_monthly_cost(instance, 'ondemand')
        monthly_reserved = get_monthly_cost(instance, 'reserved_1year')
        savings = monthly_ondemand - monthly_reserved
        savings_percent = (savings / monthly_ondemand) * 100
        
        if savings_percent > 30:  # 节省 > 30% 建议购买
            recommendations.append({
                'instance': instance,
                'roi_percent': savings_percent,
                'monthly_savings': savings,
            })
    
    return recommendations

# 3. 闲置资源审计
def find_idle_resources():
    """找出闲置/可回收资源"""
    idle = {
        'unattached_volumes': [],     # 未挂载云盘
        'unused_eips': [],            # 未绑定弹性IP
        'old_snapshots': [],          # 过期快照
        'stopped_instances': [],      # 长期停机实例
        'empty_k8s_nodes': [],        # 空 K8s 节点
    }
    
    # 未挂载云盘
    for volume in get_all_volumes():
        if volume.status == 'available' and \
           volume.last_attached_time < days_ago(7):
            idle['unattached_volumes'].append(volume)
    
    # 停机 > 30 天的实例 (建议释放)
    for instance in get_stopped_instances():
        if instance.stopped_time < days_ago(30):
            idle['stopped_instances'].append(instance)
    
    return idle
```

### Step 3: InfraCost - IaC 成本预览

```yaml
# GitHub Actions + InfraCost: PR 时自动计算成本变化
# .github/workflows/infracost.yml
name: InfraCost Preview
on:
  pull_request:
    paths: ['terraform/**']

jobs:
  infracost:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup InfraCost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
      
      - name: InfraCost Diff
        run: |
          # 生成主分支的成本基线
          git checkout main
          cd terraform/aliyun/prod
          terraform init
          terraform plan -out tfplan.binary
          infracost breakdown \
            --path tfplan.binary \
            --format json \
            --out-file /tmp/cost-main.json
          
          # 生成 PR 分支的成本预估
          git checkout ${{ github.event.pull_request.head.sha }}
          terraform plan -out tfplan.pr.binary
          infracost breakdown \
            --path tfplan.pr.binary \
            --format json \
            --out-file /tmp/cost-pr.json
      
      - name: Post Cost Comment
        uses: infracost/actions/comment@v2
        with:
          path: terraform/aliyun/prod
          behavior: update
          # PR 评论展示:
          # ┌─────────────────────────────┐
          # │ 🟢 Monthly: -¥2,340 (15%)   │
          # │ RDS 降配: xlarge → large     │
          # │ 移除未使用 NAT Gateway       │
          # └─────────────────────────────┘
```

### Step 4: 预算告警

```hcl
# 阿里云预算管理
resource "alicloud_budget" "monthly" {
  budget_name = "monthly-cloud-budget"
  budget_type = "Cost"
  
  # 月度预算: ¥50,000
  budget_amount = 50000
  time_unit      = "Monthly"
  
  # 预算规则: 分档告警
  budget_rule {
    threshold       = 80   # 80% 告警
    threshold_type  = "Actual"
    contact_groups  = ["ops-team"]
  }
  budget_rule {
    threshold       = 100  # 100% 紧急
    threshold_type  = "Forecast"  # 预测型 (根据速率预计)
    contact_groups  = ["ops-team", "finance"]
  }
}

# AWS Budget
resource "aws_budgets_budget" "monthly" {
  name              = "aws-monthly-budget"
  budget_type       = "COST"
  limit_amount      = "5000"   # $5,000
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  
  notification {
    comparison_operator = "GREATER_THAN"
    threshold           = 80
    threshold_type      = "PERCENTAGE"
    notification_type   = "FORECASTED"
    subscriber_email_addresses = ["ops@company.com"]
  }
}
```

---

## 📊 典型降本案例

| 优化项 | 操作 | 月节省 |
|--------|------|--------|
| ECS 规格优化 | 6×xlarge → large | ¥4,800 |
| RDS 按量→包年 | 3 台 RDS | ¥1,800 |
| NAT GW 共用 | 多个VPC→单 NAT | ¥1,200 |
| SLB 无用监听 | 清理 5 个 | ¥600 |
| 抢占式实例 | 批处理节点池 | ¥2,000 |
| 快照生命周期 | 自动清理 30天+ | ¥900 |
| Redis 规格优化 | 4C→2C | ¥1,500 |
| **合计** | | **~¥12,800** |

---

## 💡 FinOps 实践原则

1. **标签策略**: 所有资源必须打上 `team/service/environment` 标签
2. **成本可见**: 每个团队每周看到自己的云支出
3. **预算前置**: 上线前通过 InfraCost 预估成本
4. **定期审计**: 每月一次资源优化审计
5. **奖惩机制**: 降本团队奖励节省额的 5%

---

## 📝 面试常见问题

**Q1: FinOps 的三个阶段？**
> Inform(可见) → Optimize(优化) → Operate(运营)。先看清成本，再优化，最后形成持续运营机制。

**Q2: 预留实例和 Savings Plans 的区别？**
> RI 锁定实例规格+AZ，折扣更高(~60%)；SP 锁定消费金额，灵活性更高(~55%)。AWS SP 推荐替代 RI。

**Q3: 如何推动各团队关注成本？**
> Showback(展示) → Chargeback(内部计费) → 建立成本归属感。Grafana 各团队 Owner 看板最有效。
