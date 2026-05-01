# 项目02：企业混合云网络架构

> **云平台**: 阿里云 + 华为云 | **难度**: ⭐⭐⭐⭐ | **工具**: VPC, 专线, VPN, SD-WAN, CEN  
> **场景**: 企业原有 IDC 机房，正在逐步上云，需要打通 IDC 与多云之间的网络

---

## 📖 项目概述

### 业务场景
某金融企业核心系统仍在自建 IDC，但新业务部署在阿里云，灾备环境部署在华为云。需要实现：
- IDC ↔ 阿里云 ↔ 华为云 三地网络互通
- 数据库主从同步跨 IDC 和云
- 专线为主、VPN 为备的高可用链路

### 学习目标
- ✅ 掌握云企业网 (CEN) 和 AWS Transit Gateway 组网
- ✅ 理解 BGP 动态路由 + 专线接入
- ✅ 学会多云网络互通架构设计
- ✅ 掌握 VPN 网关 (IPSec) 配置
- ✅ 理解混合云 DNS 解析方案

---

## 🏗️ 架构设计

```
                              ┌─────────────────────────┐
                              │    企业自建 IDC           │
                              │  10.0.0.0/16            │
                              │  ┌──────┐ ┌──────┐     │
                              │  │核心DB │ │应用服 │     │
                              │  │务器  │ │务器  │     │
                              │  └──┬───┘ └──┬───┘     │
                              └─────┼────────┼─────────┘
                                    │        │
                            ┌───────┴────────┴───────┐
                            │   专线接入 (Express       │
                            │   Connect/Direct Connect) │
                            │   BGP 动态路由            │
                            └───┬───────────────┬─────┘
                                │               │
                    ┌───────────┴──┐     ┌──────┴───────────┐
                    │  阿里云 VPC   │     │   华为云 VPC       │
                    │  172.16.0.0/16│     │   192.168.0.0/16  │
                    │              │     │                   │
                    │  ┌────────┐  │     │  ┌────────┐      │
                    │  │ 云企业网 │◄─┼─────┼─►│ 云连接  │      │
                    │  │ (CEN)  │  │     │  │ (CC)   │      │
                    │  └───┬────┘  │     │  └───┬────┘      │
                    │      │       │     │      │           │
                    │  ┌───┴────┐  │     │  ┌───┴────┐      │
                    │  │可用区A  │  │     │  │可用区A  │      │
                    │  │ K8s    │  │     │  │ K8s    │      │
                    │  │ RDS    │  │     │  │ GaussDB│      │
                    │  └────────┘  │     │  └────────┘      │
                    │              │     │                   │
                    │  ┌────────┐  │     │  ┌────────┐      │
                    │  │可用区B  │  │     │  │可用区B  │      │
                    │  │ 灾备   │  │     │  │ 灾备   │      │
                    │  └────────┘  │     │  └────────┘      │
                    └──────────────┘     └───────────────────┘
```

---

## 🔧 实施步骤

### Step 1: 阿里云 - 云企业网 (CEN) 组网

```hcl
# 创建云企业网实例
resource "alicloud_cen_instance" "main" {
  name        = "hybrid-cloud-cen"
  description = "混合云企业组网"
}

# 将 VPC 挂载到 CEN
resource "alicloud_cen_instance_attachment" "vpc_main" {
  instance_id              = alicloud_cen_instance.main.id
  child_instance_id        = alicloud_vpc.main.id
  child_instance_region_id = "cn-hangzhou"
  child_instance_type      = "VPC"
}

# 配置跨地域带宽 (如果需要)
resource "alicloud_cen_bandwidth_package" "cross_region" {
  bandwidth                  = 100  # Mbps
  geographic_region_a_id     = "China"
  geographic_region_b_id     = "China"
  bandwidth_package_name     = "cross-region-bandwidth"
}
```

### Step 2: 专线接入配置

```bash
# 1. 在阿里云创建物理专线
# 控制台 → 高速通道 → 物理专线 → 创建
# 选择接入点、端口类型 (1Gbps/10Gbps)、运营商

# 2. 创建虚拟边界路由器 (VBR)
aliyun vpc CreateVirtualBorderRouter \
  --PhysicalConnectionId pc-xxx \
  --VlanId 100 \
  --PeerGatewayIp 10.1.1.1 \
  --PeeringSubnetMask 255.255.255.252 \
  --LocalGatewayIp 10.1.1.2

# 3. 配置 BGP 对等
aliyun vpc AddBgpNetwork \
  --VbrId vbr-xxx \
  --DstCidrBlock 10.0.0.0/16  # IDC 网段
```

### Step 3: VPN 备线配置 (IPSec)

```hcl
# 阿里云 VPN 网关
resource "alicloud_vpn_gateway" "main" {
  vpc_id          = alicloud_vpc.main.id
  bandwidth       = 100
  enable_ssl      = false
  instance_charge_type = "PostPaid"
  vpn_gateway_name = "idc-vpn-gateway"
}

# 用户网关 (IDC 侧路由器)
resource "alicloud_vpn_customer_gateway" "idc" {
  ip_address  = "203.0.113.1"  # IDC 公网 IP
  name        = "idc-router"
}

# IPSec 连接
resource "alicloud_vpn_connection" "idc_to_cloud" {
  vpn_gateway_id      = alicloud_vpn_gateway.main.id
  customer_gateway_id = alicloud_vpn_customer_gateway.idc.id
  local_subnet        = ["172.16.0.0/16"]  # 阿里云 VPC 网段
  remote_subnet       = ["10.0.0.0/16"]    # IDC 网段
  effect_immediately  = true
  ike_config {
    ike_auth_alg  = "sha256"
    ike_enc_alg   = "aes256"
    ike_version   = "ikev2"
    ike_mode      = "main"
    ike_lifetime  = 86400
    psk           = var.vpn_pre_shared_key
    ike_pfs       = "group14"
    ike_remote_id = "203.0.113.1"
    ike_local_id  = alicloud_vpn_gateway.main.vswitch_id
  }
  ipsec_config {
    ipsec_pfs      = "group14"
    ipsec_enc_alg  = "aes256"
    ipsec_auth_alg = "sha256"
    ipsec_lifetime = 86400
  }
}
```

### Step 4: 华为云云连接 (Cloud Connect) 组网

```hcl
# 华为云提供方 (通过 Terraform)
resource "huaweicloud_vpc" "dr" {
  name = "dr-vpc"
  cidr = "192.168.0.0/16"
}

resource "huaweicloud_vpc_subnet" "dr_subnet" {
  name       = "dr-subnet"
  cidr       = "192.168.1.0/24"
  vpc_id     = huaweicloud_vpc.dr.id
  gateway_ip = "192.168.1.1"
}

# VPN 连接阿里云到华为云
resource "huaweicloud_vpn_gateway" "to_aliyun" {
  name                = "to-aliyun-vpn"
  vpc_id              = huaweicloud_vpc.dr.id
  bandwidth           = 100
  flavor              = "Professional1"
  availability_zones  = ["cn-south-1a", "cn-south-1b"]
}

resource "huaweicloud_vpn_connection" "aliyun" {
  name                = "huawei-to-aliyun"
  gateway_id          = huaweicloud_vpn_gateway.to_aliyun.id
  gateway_ip          = huaweicloud_vpn_gateway.to_aliyun.master_eip[0].ip_address
  vgw_ip              = alicloud_vpn_gateway.main.internet_ip
  psk                 = var.vpn_pre_shared_key_huawei
  peer_subnets        = ["172.16.0.0/16"]  # 阿里云网段
  tunnel_local_address = "169.254.70.1/30"
  tunnel_peer_address  = "169.254.70.2/30"
}
```

### Step 5: 混合云 DNS 方案

```
┌──────────────────────────────────────────────────────────┐
│                   私有 DNS 解析架构                        │
│                                                           │
│   ┌─────────┐     ┌──────────┐     ┌──────────┐         │
│   │  IDC DNS │────►│阿里云 DNS│◄────►│华为云 DNS│         │
│   │(自建)    │     │(Private   │     │(内网DNS) │         │
│   │          │     │ Zone)     │     │          │         │
│   └─────────┘     └──────────┘     └──────────┘         │
│        │                │                 │              │
│        ▼                ▼                 ▼              │
│  db.internal.idc  rds.internal.cloud  gaussdb.dr.cloud   │
│  10.0.1.100       172.16.1.200       192.168.1.100       │
└──────────────────────────────────────────────────────────┘
```

```hcl
# 阿里云 Private Zone DNS
resource "alicloud_pvtz_zone" "internal" {
  zone_name = "internal.cloud"
}

resource "alicloud_pvtz_zone_record" "rds" {
  zone_id = alicloud_pvtz_zone.internal.id
  rr      = "rds"
  type    = "A"
  value   = "172.16.1.200"
  ttl     = 60
}

# 将私有域关联到 VPC
resource "alicloud_pvtz_zone_attachment" "main" {
  zone_id = alicloud_pvtz_zone.internal.id
  vpc_ids = [alicloud_vpc.main.id]
}

# 配置转发规则 → IDC DNS
resource "alicloud_pvtz_endpoint" "outbound" {
  endpoint_name = "to-idc"
  vpc_id        = alicloud_vpc.main.id
  security_group_id = alicloud_security_group.dns.id
  # 转发 internal.idc 到 IDC DNS 服务器
}

resource "alicloud_pvtz_rule" "forward_to_idc" {
  rule_name = "forward-to-idc"
  zone_name = "internal.idc"
  type      = "OUTBOUND"
  endpoint_id = alicloud_pvtz_endpoint.outbound.id
  forward_ip {
    ip   = "10.0.1.53"  # IDC DNS 服务器
    port = 53
  }
}
```

---

## 🛡️ 高可用网络设计

### 专线 + VPN 主备方案

```
               正常状态:
    IDC ──专线(Active)──► 阿里云 CEN
    IDC ──VPN(Standby)──► 阿里云 VPN GW
    华为云 ──VPN──► 阿里云 VPN GW

               专线故障:
    IDC ──专线(Down)──► 阿里云 CEN  ✗
    IDC ──VPN(Active)──► 阿里云 VPN GW  ✓ 自动切换
    华为云 ──VPN──► 阿里云 VPN GW  ✓ (不受影响)
```

### BGP 配置要点

```bash
# IDC 侧路由器 BGP 配置 (Cisco)
router bgp 65001
  neighbor 10.1.1.2 remote-as 45104    # 阿里云 VBR
  neighbor 10.1.1.2 description ALIYUN-VBR
  network 10.0.0.0 mask 255.255.0.0
  
  # BFD 快速故障检测
  neighbor 10.1.1.2 fall-over bfd
  
  # 路由优先级: 专线 > VPN
  neighbor 10.1.1.2 route-map SET_METRIC in
!
route-map SET_METRIC permit 10
  set metric 100  # 低 metric = 高优先级
```

---

## 📊 关键网络指标

| 指标 | 专线 | VPN |
|------|------|-----|
| 带宽 | 100M-10Gbps | 最多200Mbps |
| 延迟 | <5ms (同城) | >10ms (公网) |
| 可用性 | 99.95% SLA | 99.5% SLA |
| 成本 | ¥5K-50K/月 | ¥500-2K/月 |
| 加密 | 不加密(物理隔离) | IPSec 加密 |

---

## 📝 面试常见问题

**Q1: 混合云网络如何做高可用？**
> 专线为主、VPN为备，BGP 动态路由自动切换。同城双POP接入点，跨可用区冗余。

**Q2: CEN 和 VPN 有什么区别？**
> CEN 是云厂商的骨干网互联方案，低延迟高带宽；VPN 走公网 IPSec 隧道，成本低但延迟高。

**Q3: 混合云 DNS 怎么解决？**
> 私有域解析 + 条件转发。每个云有自己的 Private Zone，通过 Outbound Endpoint 转发跨域请求。

**Q4: 如何实现多云 Kubernetes 集群网络互通？**
> 底层 CEN/VPN 打通 VPC 三层网络 + Submariner/Cilium ClusterMesh 实现 K8s 服务互通。

---

## 📚 扩展阅读

- [阿里云云企业网 CEN](https://help.aliyun.com/product/59012.html)
- [华为云云连接 CC](https://support.huaweicloud.com/cc/index.html)
- [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/)
- [BGP 最佳实践](https://tools.ietf.org/html/rfc7454)
