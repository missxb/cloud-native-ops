# 项目14：全球边缘计算与CDN

> **云平台**: 阿里云 + Cloudflare | **难度**: ⭐⭐⭐⭐ | **工具**: CDN, DCDN, EdgeRoutine, Cloudflare Workers, GA  
> **场景**: 全球业务需要低延迟(全球<100ms) + 边缘计算能力(图片处理/鉴权/API代理)

---

## 📖 项目概述

某短视频平台用户遍布全球，需要最近接入点+边缘极致加速。CDN 加速静态/动态内容，EdgeRoutine/Workers 实现边缘计算。

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                      全球边缘计算与CDN架构                             │
│                                                                      │
│  全球用户                                                            │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │                   Global Accelerator (GA)                 │        │
│  │             就近接入 → 全球骨干网 → 最近源站              │        │
│  └─────────────────────────────────────────────────────────┘        │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │           CDN 边缘节点 (全球 2800+ POP 点)               │        │
│  │                                                          │        │
│  │  ┌──────────────────────────────────────────────┐       │        │
│  │  │         边缘计算运行时                        │       │        │
│  │  │  ┌─────────────────┐  ┌─────────────────┐   │       │        │
│  │  │  │ EdgeRoutine     │  │ Cloudflare     │   │       │        │
│  │  │  │ (阿里云 ER)     │  │ Workers        │   │       │        │
│  │  │  ├─────────────────┤  ├─────────────────┤   │       │        │
│  │  │  │• 鉴权/限流      │  │• A/B 测试       │   │       │        │
│  │  │  │• 图片处理       │  │• Geo 重定向     │   │       │        │
│  │  │  │• API 代理聚合   │  │• 安全过滤       │   │       │        │
│  │  │  │• 灰度路由       │  │• Header 修改    │   │       │        │
│  │  │  └─────────────────┘  └─────────────────┘   │       │        │
│  │  └──────────────────────────────────────────────┘       │        │
│  │                                                          │        │
│  │  静态加速: HTML/CSS/JS/Images → 边缘缓存                 │        │
│  │  动态加速: API请求 → 智能选路 → 回源加速                 │        │
│  │  流媒体: 直播/点播 → 智能预取 + 协议优化                 │        │
│  └──────────────────────┬──────────────────────────────────┘        │
│                         │                                            │
│  ┌──────────────────────┴──────────────────────────────────┐        │
│  │                     源站 (多Region)                       │        │
│  │     杭州 ACK 集群        新加坡 EKS 集群                 │        │
│  │     OSS 源站              S3 源站                        │        │
│  └─────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### Step 1: 阿里云 CDN + DCDN 配置

```hcl
# 阿里云 CDN 域名配置
resource "alicloud_cdn_domain_new" "static" {
  domain_name = "static.example.com"
  cdn_type    = "web"
  scope       = "overseas"  # 全球加速
  
  sources {
    type     = "oss"
    content  = alicloud_oss_bucket.static.bucket
    port     = 443
    priority = 10
    weight   = 20
  }
  
  # HTTPS 配置
  certificate_config {
    server_certificate_status = "on"
    cert_name                 = "static-cert"
    cert_type                 = "free"  # CDN 免费证书
    force_set                 = "1"     # 强制 HTTPS
  }
  
  # 性能优化
  http2     = "on"
  ipv6      = "on"
  
  # 缓存策略
  cache_config {
    cache_content_1 {
      target      = ".jpg,.png,.gif,.webp,.svg"
      ttl         = 86400    # 图片: 24h
      weight      = 80
    }
    cache_content_2 {
      target      = ".html,.css,.js"
      ttl         = 3600     # 静态资源: 1h
      weight      = 90
    }
  }
}

# DCDN (全站加速, 适用于动态API)
resource "alicloud_dcdn_domain" "api" {
  domain_name = "api.example.com"
  scope       = "overseas"
  
  sources {
    type     = "domain"
    content  = "slb-xxx.cn-hangzhou.aliyuncs.com"
    port     = 443
    priority = 20
    weight   = 10
  }
  sources {
    type     = "domain"
    content  = "alb-sg.elb.amazonaws.com"
    port     = 443
    priority = 20
    weight   = 10
  }
  
  # 回源协议
  ssl_protocol = "on"
  check_url    = "/health"
  
  # 动态加速特性 (智能选路 + 协议栈优化)
  dynamic_acceleration = "on"
}
```

### Step 2: EdgeRoutine 边缘计算 (阿里云)

```javascript
// EdgeRoutine: 在 CDN 边缘节点执行 JavaScript
// 场景1: 图片实时处理 (缩放→裁剪→转格式→水印)

/**
 * @param {Request} req - CDN 请求对象
 * @example https://img.example.com/avatar/12345.jpg?w=200&h=200&fmt=webp
 */
async function handleRequest(req) {
    const url = new URL(req.url);
    const params = url.searchParams;
    
    // 获取回源响应
    let response = await fetch(req.url, { backend: 'OSS_SOURCE' });
    
    // 如果请求带处理参数，进行边缘处理
    if (params.has('w') || params.has('h') || params.has('fmt')) {
        const image = await response.arrayBuffer();
        
        // 通过 Wasm 调用图片处理库 (Sharp)
        const processed = await imageProcessor.process(image, {
            width: parseInt(params.get('w')) || 0,
            height: parseInt(params.get('h')) || 0,
            format: params.get('fmt') || 'auto',
            quality: parseInt(params.get('q')) || 85,
        });
        
        response = new Response(processed, {
            status: 200,
            headers: {
                'Content-Type': `image/${params.get('fmt') || 'webp'}`,
                'Cache-Control': 'public, max-age=86400',
                'X-Processed-By': 'EdgeRoutine',
            }
        });
    } else {
        // 无参数，直接返回 + 设置缓存
        response.headers.set('Cache-Control', 'public, max-age=604800');
    }
    
    return response;
}

// 注册 EdgeRoutine
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});
```

```javascript
// EdgeRoutine: 场景2 - API 网关/鉴权
/**
 * @param {Request} req
 */
async function apiGateway(req) {
    const url = new URL(req.url);
    
    // 1. JWT Token 校验 (边缘鉴权, 无需回源)
    const token = req.headers.get('Authorization')?.replace('Bearer ', '');
    if (!token) {
        return new Response(JSON.stringify({ error: 'Unauthorized' }), {
            status: 401,
            headers: { 'Content-Type': 'application/json' }
        });
    }
    
    try {
        const payload = verifyJWT(token, PUBLIC_KEY);
        
        // 2. 限流 (令牌桶算法)
        const rateLimitKey = `${payload.userId}:${url.pathname}`;
        const isAllowed = await rateLimiter.check(rateLimitKey, 100, 60); // 100次/分钟
        
        if (!isAllowed) {
            return new Response(JSON.stringify({ error: 'Rate Limited' }), {
                status: 429,
                headers: { 'Content-Type': 'application/json' }
            });
        }
        
        // 3. 注入用户信息 Header (后端无需再解析 JWT)
        const headers = new Headers(req.headers);
        headers.set('X-User-Id', payload.userId);
        headers.set('X-User-Role', payload.role);
        headers.set('X-Auth-By', 'EdgeRoutine');
        
        // 4. 智能路由: 根据用户 Region 选择最近源站
        const backend = getNearestBackend(payload.region);
        const backendReq = new Request(`https://${backend}${url.pathname}${url.search}`, {
            method: req.method,
            headers: headers,
            body: req.body,
        });
        
        return await fetch(backendReq);
        
    } catch (e) {
        return new Response(JSON.stringify({ error: 'Invalid Token' }), {
            status: 403
        });
    }
}
```

### Step 3: Cloudflare Workers (海外)

```javascript
// Cloudflare Worker: 边缘 A/B 测试 + Geo 路由
export default {
    async fetch(request, env, ctx) {
        const url = new URL(request.url);
        const country = request.cf.country;  // Cloudflare 自动注入地理位置
        
        // 1. Geo 重定向: 中国大陆 → 阿里云, 海外 → AWS
        if (country === 'CN') {
            return Response.redirect(`https://cn.example.com${url.pathname}`, 302);
        }
        
        // 2. A/B 测试: Cookie 分流
        const cookie = request.headers.get('Cookie') || '';
        let variant = 'A';
        
        if (cookie.includes('ab_test=B')) {
            variant = 'B';
        } else if (!cookie.includes('ab_test=')) {
            // 50/50 随机分配
            variant = Math.random() > 0.5 ? 'A' : 'B';
        }
        
        // 3. 注入 A/B 测试 Header
        const modifiedRequest = new Request(request, {
            cf: { ...request.cf, variant },
        });
        
        const response = await fetch(modifiedRequest);
        
        // 4. 设置 Cookie (30天)
        const newResponse = new Response(response.body, response);
        newResponse.headers.set('Set-Cookie', `ab_test=${variant}; Max-Age=2592000; Path=/`);
        newResponse.headers.set('X-AB-Variant', variant);
        
        return newResponse;
    }
};
```

### Step 4: Global Accelerator (GA)

```hcl
# 阿里云全球加速 GA
resource "alicloud_ga_accelerator" "global" {
  accelerator_name  = "global-api-gateway"
  spec              = "2"       # 中型
  duration          = 1
  pricing_cycle     = "Month"
  auto_use_coupon   = true
}

# 带宽包
resource "alicloud_ga_bandwidth_package" "main" {
  bandwidth          = 100  # 100Mbps
  type               = "Basic"
  bandwidth_type     = "Enhanced"
  duration           = 1
  pricing_cycle      = "Month"
}

# 监听器
resource "alicloud_ga_listener" "https" {
  accelerator_id = alicloud_ga_accelerator.global.id
  port_ranges {
    from_port = 443
    to_port   = 443
  }
  protocol      = "HTTPS"
  proxy_protocol = false
}

# 终端节点组: 杭州
resource "alicloud_ga_endpoint_group" "hangzhou" {
  accelerator_id     = alicloud_ga_accelerator.global.id
  listener_id        = alicloud_ga_listener.https.id
  endpoint_group_region = "cn-hangzhou"
  
  endpoint_configurations {
    endpoint = alicloud_slb.api_hz.ip_address
    type     = "SLB"
    weight   = 100
  }
}

# 终端节点组: 新加坡
resource "alicloud_ga_endpoint_group" "singapore" {
  accelerator_id     = alicloud_ga_accelerator.global.id
  listener_id        = alicloud_ga_listener.https.id
  endpoint_group_region = "ap-southeast-1"  # AWS Singapore
  
  endpoint_configurations {
    endpoint = aws_lb.api_sg.dns_name
    type     = "CustomDomain"
    weight   = 100
  }
}
```

---

## 📊 CDN vs GA 对比

| 维度 | CDN/DCDN | GA 全球加速 |
|------|----------|-------------|
| 加速层级 | L7 (HTTP/HTTPS) | L4 (TCP/UDP) + L7 |
| 适用场景 | Web/API/下载/流媒体 | 游戏/视频/物联网 |
| 就近接入 | 是 | 是 |
| 骨干网加速 | 是 | 是 |
| 边缘计算 | EdgeRoutine | 不支持 |
| 成本 | 按流量 | 按带宽 |

---

## 💰 成本估算

| 服务 | 月费 |
|------|------|
| CDN (100TB流量, 东南亚) | ~¥4,200 |
| DCDN (10TB API流量) | ~¥2,500 |
| EdgeRoutine (1000万次) | ~¥800 |
| Cloudflare Workers (1000万次, 免费配额) | ¥0 |
| GA (100Mbps) | ~¥7,000 |
| **合计** | **~¥14,500** |

---

## 📝 面试常见问题

**Q1: CDN 和 DCDN 的区别？**
> CDN 加速静态内容(缓存命中直接返回, 不回源)；DCDN 加速动态内容(智能选路+协议优化, 需要回源但通过最优路径)。

**Q2: EdgeRoutine 有什么用？**
> 在 CDN 边缘节点执行代码(JS)，无需回源即可完成：鉴权/限流/图片处理/A/B测试/API聚合。延迟从 ~200ms(回源) 降到 ~5ms(边缘)。

**Q3: GA 的加速原理？**
> 1. 就近接入 (EndPoint) 2. 全球骨干网传输 3. 协议优化(TCP优化/多路复用) 4. 就近回源。延迟可降低 60-80%。
