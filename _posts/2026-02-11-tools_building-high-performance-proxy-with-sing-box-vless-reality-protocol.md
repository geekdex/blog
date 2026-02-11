---
title: 使用 sing-box 自建高性能科学上网的快速方案
date: 2026-02-11 12:26:00 +0800
categories: [工具]
tags: [网络工具, 正向代理]
pin: false
---

## 为什么选择 Sing-Box？

- **性能优越**：Golang 编写，内存占用低，处理速度快
- **抗封锁强**：VLESS-Reality 协议难以被检测和封锁
- **配置灵活**：支持分流规则、DNS 智能解析、多协议兼容
- **开源透明**：MIT 协议，社区活跃，持续更新

---

## 一、服务端配置

### 1.1 安装 Sing-Box

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

### 1.2 生成密钥

依次执行以下命令，保存输出结果：

```bash
# 生成密钥对
sing-box generate reality-keypair
# 输出：PrivateKey 和 PublicKey

# 生成 UUID
sing-box generate uuid

# 生成 Short ID
sing-box generate rand --hex 8
```

### 1.3 创建配置文件

编辑 `/etc/sing-box/config.json`：

```json
{
  "log": { "level": "info" },
  "inbounds": [{
    "type": "vless",
    "listen": "::",
    "listen_port": 443,
    "users": [{
      "uuid": "YOUR_UUID",
      "flow": "xtls-rprx-vision"
    }],
    "tls": {
      "enabled": true,
      "server_name": "www.microsoft.com",
      "reality": {
        "enabled": true,
        "handshake": {
          "server": "www.microsoft.com",
          "server_port": 443
        },
        "private_key": "YOUR_PRIVATE_KEY",
        "short_id": ["YOUR_SHORT_ID"]
      }
    }
  }],
  "outbounds": [{
    "type": "direct"
  }]
}
```

### 1.4 启动服务

```bash
# 检查配置
sing-box check -c /etc/sing-box/config.json

# 启动并设置开机自启
systemctl enable --now sing-box

# 查看运行状态
systemctl status sing-box
```

**重要提示**：

- 确保防火墙开放 443 端口
- 伪装域名 `www.microsoft.com` 可替换为其他支持 TLS 1.3 的网站

---

## 二、客户端配置

### 2.1 配置理念：精准控制优先

**为什么需要手动指定规则？**

GeoSite 和 GeoIP 规则集虽然覆盖广泛，但无法满足所有场景：

- 某些新域名未收录到规则集
- 企业内网域名需要特殊处理
- 特定服务需要强制走代理或直连
- 某些 IP 段需要自定义路由

**最佳实践**：手动规则 + 规则集组合，实现精准控制。

### 2.2 配置文件结构

客户端配置包含四个核心部分：

1. **DNS 解析**：分流域名到不同 DNS 服务器
2. **入站 (Inbounds)**：TUN 模式接管系统流量
3. **出站 (Outbounds)**：代理和直连配置
4. **路由规则 (Route)**：决定流量走向

### 2.3 完整配置文件（含手动规则示例）

创建客户端配置文件 `config.json`：

```json
{
  "log": { "level": "info" },
  "dns": {
    "servers": [
      { "tag": "google", "address": "8.8.8.8" },
      { "tag": "cloudflare", "address": "1.1.1.1" },
      { "tag": "local", "address": "192.168.9.253", "detour": "direct" }
    ],
    "rules": [
      {
        "domain_suffix": [
          "company.internal",
          "nas.local"
        ],
        "server": "local"
      },
      {
        "domain_suffix": [
          "baidu.com",
          "taobao.com",
          "qq.com",
          "aliyun.com",
          "bilibili.com"
        ],
        "server": "local"
      },
      {
        "domain_suffix": [
          "google.com",
          "youtube.com",
          "github.com",
          "openai.com",
          "anthropic.com",
          "cloudflare.com"
        ],
        "server": "google"
      },
      {
        "rule_set": "geosite-cn",
        "server": "local"
      }
    ],
    "final": "google"
  },
  "inbounds": [{
    "type": "tun",
    "mtu": 9000,
    "address": ["172.19.0.1/30"],
    "auto_route": true,
    "strict_route": true,
    "sniff": true,
    "sniff_override_destination": true
  }],
  "outbounds": [{
    "type": "vless",
    "tag": "proxy",
    "server": "YOUR_SERVER_IP",
    "server_port": 443,
    "uuid": "YOUR_UUID",
    "flow": "xtls-rprx-vision",
    "tls": {
      "enabled": true,
      "server_name": "www.microsoft.com",
      "utls": { "enabled": true, "fingerprint": "chrome" },
      "reality": {
        "enabled": true,
        "public_key": "YOUR_PUBLIC_KEY",
        "short_id": "YOUR_SHORT_ID"
      }
    }
  }, {
    "type": "direct",
    "tag": "direct"
  }],
  "route": {
    "rules": [
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "ip_cidr": [
          "10.0.0.0/8",
          "172.16.0.0/12",
          "192.168.0.0/16"
        ],
        "outbound": "direct"
      },
      {
        "domain_suffix": [
          "company.internal",
          "nas.local"
        ],
        "outbound": "direct"
      },
      {
        "domain_suffix": [
          "baidu.com",
          "taobao.com",
          "qq.com",
          "aliyun.com",
          "bilibili.com",
          "zhihu.com",
          "douban.com"
        ],
        "outbound": "direct"
      },
      {
        "domain_suffix": [
          "google.com",
          "youtube.com",
          "github.com",
          "githubusercontent.com",
          "openai.com",
          "anthropic.com",
          "cloudflare.com",
          "twitter.com",
          "facebook.com",
          "instagram.com"
        ],
        "outbound": "proxy"
      },
      {
        "ip_cidr": [
          "1.1.1.0/24",
          "8.8.8.0/24"
        ],
        "outbound": "proxy"
      },
      {
        "rule_set": "geosite-cn",
        "outbound": "direct"
      },
      {
        "rule_set": "geoip-cn",
        "outbound": "direct"
      }
    ],
    "rule_set": [
      {
        "tag": "geosite-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs",
        "download_detour": "direct"
      },
      {
        "tag": "geoip-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs",
        "download_detour": "direct"
      }
    ],
    "final": "proxy",
    "auto_detect_interface": true
  }
}
```

### 2.4 规则配置详解

#### DNS 规则（优先级从上到下）

```json
"dns": {
  "rules": [
    // 1. 内网域名 → 本地 DNS
    { "domain_suffix": ["company.internal", "nas.local"], "server": "local" },
    
    // 2. 国内域名 → 本地 DNS（避免污染）
    { "domain_suffix": ["baidu.com", "qq.com"], "server": "local" },
    
    // 3. 国外域名 → Google DNS
    { "domain_suffix": ["google.com", "youtube.com"], "server": "google" },
    
    // 4. 规则集兜底
    { "rule_set": "geosite-cn", "server": "local" }
  ],
  "final": "google"  // 默认使用 Google DNS
}
```

#### 路由规则（优先级从上到下）

```json
"route": {
  "rules": [
    // 1. 私有 IP → 直连（内网流量）
    { "ip_is_private": true, "outbound": "direct" },
    
    // 2. 指定 IP 段 → 直连（自定义网段）
    { "ip_cidr": ["10.0.0.0/8", "192.168.0.0/16"], "outbound": "direct" },
    
    // 3. 内网域名 → 直连
    { "domain_suffix": ["company.internal"], "outbound": "direct" },
    
    // 4. 国内域名 → 直连（手动指定）
    { "domain_suffix": ["baidu.com", "qq.com"], "outbound": "direct" },
    
    // 5. 国外域名 → 代理（手动指定）
    { "domain_suffix": ["google.com", "youtube.com"], "outbound": "proxy" },
    
    // 6. 指定 IP → 代理（如 DNS 服务器）
    { "ip_cidr": ["8.8.8.0/24"], "outbound": "proxy" },
    
    // 7. 规则集兜底
    { "rule_set": "geosite-cn", "outbound": "direct" },
    { "rule_set": "geoip-cn", "outbound": "direct" }
  ],
  "final": "proxy"  // 默认走代理
}
```

### 2.5 关键参数说明

| 参数 | 说明 |
|------|------|
| `server` | 替换为你的服务器 IP 地址 |
| `uuid` | 必须与服务端完全一致 |
| `public_key` | 服务端生成的 PublicKey |
| `short_id` | 必须与服务端完全一致 |
| `utls` | 伪装成 Chrome 浏览器的 TLS 指纹 |
| `domain_suffix` | 匹配域名后缀，如 `google.com` 可匹配 `www.google.com` |
| `ip_cidr` | CIDR 格式的 IP 段，如 `192.168.0.0/16` |
| `final` | 所有规则都不匹配时的默认行为 |

### 2.6 自定义规则示例

根据实际需求调整以下规则：

**场景 1：企业内网访问**

```json
{
  "domain_suffix": ["company.local", "intranet.example.com"],
  "outbound": "direct"
}
```

**场景 2：特定服务强制代理**

```json
{
  "domain_suffix": ["openai.com", "gemini.google.com"],
  "outbound": "proxy"
}
```

**场景 3：CDN 节点直连**

```json
{
  "ip_cidr": ["104.16.0.0/12", "172.64.0.0/13"],
  "outbound": "direct"
}
```

**场景 4：游戏服务器加速**

```json
{
  "domain_suffix": ["steamcommunity.com", "discord.com"],
  "outbound": "proxy"
}
```

### 2.7 规则匹配逻辑

**处理流程**：DNS 解析 → 获得 IP → 路由匹配 → 选择出站

**匹配顺序**：

1. 规则从上到下依次匹配
2. 命中第一条规则后停止，不再继续匹配
3. 所有规则都不匹配时，使用 `final` 指定的默认出站

**最佳实践**：

- 手动规则放在前面（高优先级）
- 规则集放在后面（兜底）
- `final` 根据使用习惯设置（proxy 或 direct）

---

## 三、常见问题

### 3.1 连接失败排查

- 检查服务器防火墙是否开放 443 端口
- 确认 uuid、public_key、short_id 与服务端一致
- 查看服务端日志：`journalctl -u sing-box -f`
- 验证伪装域名可访问且支持 TLS 1.3

### 3.2 分流不生效

- DNS 解析优先于路由规则，先检查 DNS 配置
- 确保 GeoSite/GeoIP 规则集下载成功

### 3.3 服务管理命令

```bash
# 重启服务
systemctl restart sing-box

# 查看日志
journalctl -u sing-box -n 50 --no-pager

# 测试配置
sing-box check -c /etc/sing-box/config.json
```

---

*配置完成后即可享受高速、稳定的网络连接*
