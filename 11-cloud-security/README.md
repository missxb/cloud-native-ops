# 项目11：云安全纵深防御体系

> **云平台**: 阿里云 + 天翼云 | **难度**: ⭐⭐⭐⭐ | **工具**: WAF, CSPM, KMS, 态势感知, 堡垒机, 等保合规  
> **场景**: 政务/金融企业上云，需满足等保2.0三级 + 行业安全合规

---

## 📖 项目概述

某政务系统迁移到阿里云(主) + 天翼云(灾备)，要求满足等保2.0三级，构建纵深防御体系：网络→主机→应用→数据 四层防护。

### 学习目标
- ✅ 等保2.0三级技术要求落地
- ✅ WAF + DDoS高防 + 云防火墙
- ✅ KMS 密钥管理 + 数据加密
- ✅ 态势感知 + 安全中心 CSPM
- ✅ 堡垒机 + 操作审计

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                     纵深防御安全架构 (等保2.0三级)                      │
│                                                                      │
│  互联网 → ┌──────────────────────────────────┐                      │
│          │  第0层: DDoS高防 + WAF + CDN       │                      │
│          │  (流量清洗 + Web攻击拦截)           │                      │
│          └────────────────┬─────────────────┘                      │
│                           ▼                                         │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  第1层: 网络边界防护 (VPC安全组 + 云防火墙 + NAT)        │        │
│  │  ┌──────────────────────────────────────────────────┐ │        │
│  │  │  SLB (仅开放443) → WAF联动                        │ │        │
│  │  │  DMZ区 (堡垒机/代理)                               │ │        │
│  │  └────────┬─────────────────────────────────────────┘ │        │
│  └───────────┼───────────────────────────────────────────┘        │
│              ▼                                                     │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  第2层: 主机安全 (云安全中心 + 安骑士)                    │        │
│  │  • 漏洞扫描与管理                                        │        │
│  │  • 异常登录/暴力破解检测                                  │        │
│  │  • 病毒查杀 + 基线检查                                    │        │
│  └───────────┬────────────────────────────────────────────┘        │
│              ▼                                                     │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  第3层: 应用安全 (RASP + API 安全网关)                    │        │
│  │  • SQL注入/XSS 防护                                      │        │
│  │  • API 限流 + 认证                                       │        │
│  │  • JWT 验证 + RBAC                                       │        │
│  └───────────┬────────────────────────────────────────────┘        │
│              ▼                                                     │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  第4层: 数据安全 (KMS + 加密 + 审计)                      │        │
│  │  • 传输加密 (TLS 1.3)                                    │        │
│  │  • 存储加密 (云盘/OSS/RDS)                                │        │
│  │  • 密钥管理 (KMS + 自动轮转)                              │        │
│  │  • 操作审计 (ActionTrail/CloudTrail)                      │        │
│  │  • 数据库审计 (DBAudit)                                   │        │
│  └────────────────────────────────────────────────────────┘        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────┐        │
│  │  统一安全管理: 态势感知 (SIEM) + 安全中心 (CSPM)          │        │
│  │  • 安全态势大屏                                           │        │
│  │  • 合规检查 (等保/CIS/ISO27001)                           │        │
│  │  • 自动化安全编排与响应 (SOAR)                            │        │
│  └────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: WAF + DDoS 配置

```hcl
# 阿里云 WAF 3.0 实例
resource "alicloud_wafv3_instance" "main" {
  # WAF 3.0 按量付费
  # 包含: 基础Web防护 + Bot管理 + API安全
  instance_name = "webapp-waf"
}

# WAF 防护域名
resource "alicloud_wafv3_domain" "webapp" {
  instance_id = alicloud_wafv3_instance.main.id
  domain      = "api.example.com"
  listen = {
    https = {
      port = 443
    }
  }
  redirect = {
    backends = ["slb-xxx.cn-hangzhou.aliyuncs.com"]
  }
  
  # HTTPS 证书
  cert_id = alicloud_ssl_certificate.webapp.id
  
  # 日志投递 SLS
  log_headers = [
    { key = "X-User-Id" },
    { key = "X-Request-Id" },
  ]
}

# DDoS 高防 (针对金融/游戏类)
resource "alicloud_ddos_bgp_instance" "main" {
  base_bandwidth = 20          # 基础防护 20Gbps
  bandwith       = 100          # 弹性防护 100Gbps
  ip_type        = "IPv4"
  ip_count       = 2
}
```

### Step 2: 云防火墙 + 安全组策略

```hcl
# 阿里云云防火墙 (CFW)
resource "alicloud_cloud_firewall_instance" "main" {
  ip_number       = 50
  spec            = "premium_version"  # 企业版
  bandwidth       = 100  # Mbps
  log_storage     = 100  # GB
}

# VPC 安全组: 最小权限原则
resource "alicloud_security_group" "web" {
  name        = "web-tier-sg"
  vpc_id      = alicloud_vpc.main.id
  description = "Web 层安全组 (仅允许必要入口)"
}

# 精细安全组规则
resource "alicloud_security_group_rule" "allow_https" {
  type              = "ingress"
  ip_protocol       = "tcp"
  port_range        = "443/443"
  security_group_id = alicloud_security_group.web.id
  cidr_ip           = "0.0.0.0/0"
  description       = "HTTPS public access"
}

resource "alicloud_security_group_rule" "allow_waf_backend" {
  type              = "ingress"
  ip_protocol       = "tcp"
  port_range        = "8443/8443"
  security_group_id = alicloud_security_group.web.id
  # 仅允许 WAF 回源 IP
  source_security_group_id = alicloud_security_group.waf.id
  description              = "WAF backend only"
}

# 拒绝所有出公网流量 (仅允许必要的外网 API)
resource "alicloud_security_group_rule" "deny_egress_default" {
  type              = "egress"
  ip_protocol       = "all"
  port_range        = "-1/-1"
  security_group_id = alicloud_security_group.app.id
  cidr_ip           = "0.0.0.0/0"
  policy            = "drop"
  description       = "Default deny all outbound"
}
```

### Step 3: KMS 密钥管理 + 数据加密

```hcl
# KMS 密钥: 应用数据库加密密钥
resource "alicloud_kms_key" "db_encryption" {
  description = "RDS/Redis encryption key"
  key_usage   = "ENCRYPT/DECRYPT"
  
  # 自动轮转 (90天)
  rotation_interval_days = 90
  
  # 密钥策略: 仅允许授权服务使用
  policy = jsonencode({
    Version = "1",
    Statement = [
      {
        Sid    = "AllowAppService",
        Effect = "Allow",
        Principal = {
          RAM = [
            "acs:ram::123456:role/rds-encryption-role",
          ]
        },
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:GenerateDataKey",
        ],
        Resource = ["*"]
      }
    ]
  })
}

# RDS 加密
resource "alicloud_db_instance" "main" {
  engine               = "MySQL"
  engine_version       = "8.0"
  instance_type        = "mysql.x4.xlarge"
  
  # 启用 KMS 加密
  encryption           = "CloudDisk"
  encryption_key       = alicloud_kms_key.db_encryption.id
  
  # SSL 强制
  ssl_action           = "Open"
  force_ssl            = "true"
}

# OSS 加密 (AES256 + KMS)
resource "alicloud_oss_bucket" "sensitive" {
  bucket = "sensitive-data-bucket"
  
  server_side_encryption_rule {
    sse_algorithm     = "KMS"
    kms_master_key_id = alicloud_kms_key.oss_encryption.id
  }
  
  # 公共访问禁止
  acl = "private"
  
  # 版本控制 (防勒索/误删)
  versioning {
    status = "Enabled"
  }
}
```

### Step 4: 堡垒机 + 审计 (天翼云)

```hcl
# 天翼云 (CTYun) 堡垒机 + 安全审计
# 天翼云提供安全专区方案，适用政务云场景

# 云堡垒机 (CBH)
# 通过天翼云控制台或 API 创建

# 操作审计 (ActionTrail)
# 记录所有控制台 + API 操作
# 控制台 → 云审计 → 创建跟踪
# 投递到 OOS (天翼云对象存储)
```

### Step 5: 安全编排自动化响应 (SOAR)

```python
# 安全事件自动化处理脚本 (Python + 阿里云 SDK)
# 场景: WAF 检测到暴力破解 → 自动封禁IP + 通知

import json
from aliyunsdkcore.client import AcsClient
from aliyunsdkwaf_openapi.request.v20190910 import ModifyProtectionRuleRequest

def lambda_handler(event, context):
    """WAF 告警触发 → 自动化响应"""
    
    alert = json.loads(event['Records'][0]['Sms']['body'])
    
    # 1. WAF 检测到高频访问
    if alert['ruleId'] == 'high_frequency_access':
        attacker_ip = alert['detail']['remoteAddr']
        
        # 2. 自动加入 WAF 黑名单 (封禁24小时)
        client = AcsClient(ACCESS_KEY, SECRET_KEY, 'cn-hangzhou')
        request = ModifyProtectionRuleRequest()
        request.set_Domain('api.example.com')
        request.set_RuleId(1001)
        request.set_LockVersion(0)
        request.set_Config({
            'action': 'block',
            'blockedIps': [attacker_ip],
            'duration': 86400  # 24h
        })
        client.do_action_with_exception(request)
        
        # 3. 同步到安全组黑名单
        # add_to_security_group_blacklist(attacker_ip)
        
        # 4. 通知安全团队
        send_dingtalk_alert({
            'title': '🛡️ 自动封禁恶意IP',
            'ip': attacker_ip,
            'action': 'WAF黑名单 (24h)',
            'timestamp': alert['timestamp']
        })
        
        return {'status': 'blocked', 'ip': attacker_ip}
```

---

## 📊 等保2.0三级核心要求对照

| 安全层面 | 关键要求 | 云上实现 |
|----------|----------|----------|
| 网络安全 | 访问控制、入侵防范 | VPC+SG+NACL+云防火墙+WAF |
| 主机安全 | 身份鉴别、安全审计 | 堡垒机+安骑士+安全中心 |
| 应用安全 | 安全审计、通信完整性 | WAF+API网关+TLS1.3 |
| 数据安全 | 数据加密、备份恢复 | KMS+RDS加密+DBS备份 |
| 安全管理 | 集中管控、审计追踪 | 态势感知+ActionTrail+SOC |

---

## 💰 安全成本估算

| 服务 | 月费 (¥) |
|------|----------|
| WAF 3.0 (企业版) | ¥29,800 |
| 云防火墙 (企业版) | ¥4,200 |
| DDoS高防 (20Gbps) | ¥20,800 |
| 云安全中心 (企业版) | ¥600 |
| KMS (密钥管理) | ¥0 (按调用付费，~¥200) |
| 堡垒机 (20资产) | ¥1,800 |
| 态势感知 | ¥15,000 |
| 数据库审计 | ¥3,000 |
| **合计** | **~¥75,400** |

> ⚠️ 安全是刚性成本，尤其金融/政务行业

---

## 📝 面试常见问题

**Q1: 等保2.0三级的核心变化？**
> 新增云计算安全扩展要求、移动互联安全扩展要求、物联网安全扩展要求。强调"一个中心(安全管理中心) + 三重防护(计算环境+区域边界+通信网络)"。

**Q2: 云上安全责任共担模型？**
> 云厂商负责"云的安全"(物理/虚拟化/网络)，用户负责"云中的安全"(数据/应用/身份/配置)。

**Q3: WAF 和云防火墙的区别？**
> WAF 是7层应用防护(SQL注入/XSS/CC攻击)，云防火墙是4层网络防护(IP/端口/协议) + 东西向流量隔离。
