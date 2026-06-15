---
title: 我用 sing-box 打造了一套极致精准的网络流量调度网关（全平台通用+策略路由）
date: 2026-06-06 10:50:00 +0800
categories: [网络]
tags: [网关]
pin: false
---

## 一、 工具介绍与核心工作流

在尝试了诸多网络转发工具与繁杂的第三方 GUI 面板后，我最终将整套网络环境的路由方案迁移到了纯粹的 **sing-box** 官方原版核心。

这套配置的设计哲学围绕以下三大核心理念展开：

1. **拒绝黑盒，精准控流**：通过详尽的路由规则，掌握系统里每一个数据包的走向，绝不让流量不明不白地走错节点。
2. **极简配置与直连兜底**：遵循极其严谨的优先级：**固定 IP > IP 范围 (CIDR) > 固定域名 > 域名后缀 > Geosite 数据库**。在日常网络环境下，我们采用**“最终使用直连（direct）做兜底”**的白名单策略。遇到访问缓慢或解析异常的海外技术资源（如 GitHub、Docker Hub 等）再按需分流，这是降低延迟、最节省节点带宽的做法。
3. **全平台统一与网关化**：只使用 sing-box 官方原版核心，一份通用的 JSON 配置吃透 Linux、Windows、macOS、Android 和 iOS。在 Linux 设备上，只需开启 IP 转发，它还能瞬间化身为局域网的透明流量网关。

在动手配置之前，我们必须先理解手里的武器是什么。

### 1.1 什么是 sing-box？

sing-box 是一个现代化的网络路由与隧道工具，主要用于网络分流、跨节点通信和内网穿透。它不是传统的单一服务，而是一个**高性能的流量调度器 + 多协议实现引擎**。

换一句话说：**sing-box 可以根据你设定的规则，极其精准地把不同的网络流量动态路由到不同的出口（直连或远程隧道）。**

### 1.2 为什么选择 sing-box？

- **性能优越**：Golang 编写，内存占用极低，高并发下处理速度极快。
- **配置灵活**：支持极为强大的分流规则、DNS 智能解析机制以及多协议兼容。
- **开源透明**：采用 MIT 协议，社区活跃，代码透明，便于二次开发与深度定制。

### 1.3 sing-box 的执行流（管道模型）

理解 sing-box 配置其实就是理解一个数据流管道模型。最核心的三个概念是：

```text
入口 (inbound) → 路由 (route) → 出口 (outbound)
```

再加上一个 **DNS 系统** 在旁边参与决策。整体执行流程如下：

```text
启动
 ↓
加载配置
 ↓
启动 DNS 模块
 ↓
启动 inbounds (监听本地请求)
 ↓
接收连接 (接收流量)
 ↓
route.rules 从上到下逐条匹配（遇到第一条命中的规则即停止）
 ↓
选择 outbound (决定是本地直连还是走远程隧道)
 ↓
发送

```

**关键认知：route.rules 是自上而下、先匹配先生效的。** 规则的顺序至关重要，位置靠前的规则优先级更高。

### 1.4 了解加密隧道协议

为了实现跨区域服务器之间的安全通信，我们的底层通道使用的是轻量级的高级隧道协议。此类协议的设计目标是：**极低的握手延迟、减少协议冗余开销，同时通过高度加密降低流量被中间网络节点进行 QoS（服务质量）限速或干扰的概率。**

**完整流程**：

1. 客户端请求某个外部资源（比如拉取海外 Docker 镜像）。
2. 请求数据通过隧道协议进行底层封装。
3. 通过 TLS 加密传输，伪装成普通的 HTTPS 网页浏览流量。
4. 远程服务器接收解密后，代为获取目标资源。
5. 服务器把响应数据加密返回给客户端。

### 1.5 我们的实现目标

接下来我们将基于高性能隧道协议实现最稳健的网络环境配置。下文的全套流程使用 `sing-box 1.13.3` 成功验证，在使用过程中，请根据实际业务环境与需求进行替换修改。

---

## 二、 服务端配置：构建安全通信节点

服务端我们采用目前稳定性和隐蔽性极佳的组合：**VLESS + XTLS-Vision + Reality**。通过复用 `www.bing.com` 等常规网站的 TLS 特征，极大提升通信通道的生存能力。

### 2.1 安装与环境准备

在你的远程服务器（VPS）上执行官方一键安装脚本：

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)

```

接着，我们需要生成专属的连接凭证。使用 `sing-box generate` 命令生成相关密钥：

```bash
# 生成 Reality 密钥对
sing-box generate reality-keypair
# 注意：务必将屏幕上输出的 PrivateKey 和 PublicKey 复制并妥善保存

# 生成 UUID
sing-box generate uuid

# 生成 Short ID
sing-box generate rand --hex 8

```

### 2.2 服务端配置详解 (`/etc/sing-box/config.json`)

将以下配置写入服务端的配置文件中（请将 `uuid`、`private_key` 和 `short_id` 替换为你刚才生成的值）：

```json
{
  "log": {
    "disabled": false,
    "level": "warn",
    "output": "/var/log/sing-box.log",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "type": "local",
        "tag": "local-dns"
      }
    ],
    "strategy": "prefer_ipv4"
  },
  "inbounds": [
    {
      "type": "vless",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "uuid": "YOUR_UUID",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.bing.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.bing.com",
            "server_port": 443
          },
          "private_key": "YOUR_PRIVATE_KEY",
          "short_id": ["YOUR_SHORT_ID"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct",
      "domain_resolver": "local-dns"
    }
  ],
  "route": {
    "default_domain_resolver": "local-dns"
  }
}
```

**精细解析：**

- **XTLS-Vision**：能够识别出加密流量里的数据包长度特征并进行底层填充，有效防止复杂的深度包检测（DPI）干扰。
- **Reality**：直接复用 `www.bing.com` 的握手响应。对于外部探测或中间节点来说，这看起来就像是普通的网页浏览请求，没有任何异常隧道迹象。
- **`domain_resolver`**：服务端 direct outbound 显式指定 DNS，确保域名解析行为可控。

---

## 三、 客户端配置：全平台通用的智能调度大脑

客户端配置是这套系统的核心。以下是完整的示例配置，**你可以直接将它保存在任何平台的 sing-box 客户端中使用**：

```json
{
  "log": {
    "disabled": false,
    "level": "warn",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "proxy-dns",
        "type": "tls",
        "server": "8.8.8.8",
        "detour": "proxy"
      },
      {
        "tag": "local-dns",
        "type": "local"
      }
    ],
    "rules": [
      {
        "domain": ["cn.bing.com"],
        "server": "local-dns"
      },
      {
        "domain_suffix": [
          ".cn",
          "qq.com",
          "tencent.com",
          "deepseek.com",
          "aliyuncs.com",
          "qcloud.com"
        ],
        "server": "local-dns"
      },
      {
        "domain_suffix": [
          ".google",
          ".io",
          ".dev",
          ".ai",
          ".app",
          "github.com",
          "githubusercontent.com",
          "claude.com",
          "chatgpt.com"
        ],
        "server": "proxy-dns"
      },
      {
        "rule_set": "geosite-cn",
        "server": "local-dns"
      },
      {
        "rule_set": [
          "geosite-google",
          "geosite-youtube",
          "geosite-github",
          "geosite-openai",
          "geosite-apple",
          "geosite-jetbrains"
        ],
        "server": "proxy-dns"
      }
    ],
    "final": "local-dns",
    "independent_cache": true,
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "address": ["172.19.0.1/30"],
      "mtu": 9000,
      "auto_route": true,
      "strict_route": true,
      "platform": {
        "http_proxy": {
          "enabled": true,
          "server": "127.0.0.1",
          "server_port": 2334
        }
      }
    },
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "::",
      "listen_port": 2334
    }
  ],
  "outbounds": [
    {
      "type": "vless",
      "tag": "proxy",
      "server": "YOUR_VPS_IP",
      "server_port": 443,
      "uuid": "YOUR_UUID",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.bing.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "YOUR_PUBLIC_KEY",
          "short_id": "YOUR_SHORT_ID"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "default_domain_resolver": "local-dns",
    "rule_set": [
      {
        "tag": "geosite-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geoip-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geosite-google",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-google.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geosite-github",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-github.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geosite-openai",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-openai.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geosite-jetbrains",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-jetbrains.srs",
        "download_detour": "proxy"
      }
    ],
    "rules": [
      {
        "ip_cidr": ["YOUR_VPS_IP/32"],
        "outbound": "direct"
      },
      {
        "type": "logical",
        "mode": "or",
        "rules": [{ "protocol": "dns" }, { "port": 53 }],
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "domain": ["cn.bing.com"],
        "outbound": "direct"
      },
      {
        "domain_suffix": [
          ".cn",
          "qq.com",
          "tencent.com",
          "deepseek.com",
          "aliyuncs.com",
          "qcloud.com"
        ],
        "outbound": "direct"
      },
      {
        "domain_suffix": [
          ".google",
          ".io",
          ".dev",
          ".ai",
          ".app",
          "github.com",
          "githubusercontent.com",
          "claude.com",
          "chatgpt.com"
        ],
        "outbound": "proxy"
      },
      {
        "rule_set": ["geoip-cn", "geosite-cn"],
        "outbound": "direct"
      },
      {
        "rule_set": [
          "geosite-google",
          "geosite-github",
          "geosite-openai",
          "geosite-jetbrains"
        ],
        "outbound": "proxy"
      }
    ],
    "final": "direct",
    "auto_detect_interface": true
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_rdrc": true
    }
  }
}
```

客户端配置文件极具极客美学，我们将其拆解为四大核心模块来进行**精细解析**：

### 3.1 路由规则顺序（核心：理解为何这样排列）

这是整套配置最关键、也最容易踩坑的部分。sing-box 的 `route.rules` **自上而下匹配，遇到第一条命中的规则立即执行，不再继续向下**。因此规则的顺序决定一切。

**第一条：远程服务器 IP 直连**

```json
{ "ip_cidr": ["YOUR_VPS_IP/32"], "outbound": "direct" }
```

隧道节点本身的流量必须直连，否则会形成死循环。这是所有规则中优先级最高的一条。

**第二条：用端口 53 接管 DNS 流量（关键！）**

```json
{
  "type": "logical",
  "mode": "or",
  "rules": [{ "protocol": "dns" }, { "port": 53 }],
  "action": "hijack-dns"
}
```

这是整套配置的核心。为什么要用 `port: 53` 而不是只用 `protocol: dns`？

因为 `protocol: dns` 依赖 `sniff` 动作来识别协议，而 `sniff` 在后面。如果 DNS 接管动作放在 `sniff` 之后，那么发往网关 IP（如局域网内的 `192.168.1.1`）的 DNS 查询包，会先被第三条 `ip_is_private` 规则命中并直接放行，sing-box 就永远无法介入 DNS 解析阶段。

**第三条：私网 IP 直连（必须在 sniff 之前）**

```json
{ "ip_is_private": true, "outbound": "direct" }
```

此规则让所有内网流量（如连接本地 Redis 或数据库 `10.x.x.x`）直接放行，**完全跳过后面的 sniff**。这是保证内网微服务交互和数据库连接零额外延迟的关键。

由于 `sniff` 默认会等待 300ms 读取第一个数据包来识别协议。数据库（如 MySQL）是服务端先发包，如果进入 sniff 会白白等待超时。内网 IP 提前命中直连，完美解决此延迟痛点。

**第四条：sniff 协议嗅探**

```json
{ "action": "sniff" }
```

对所有到达此处的流量提取出连接目标的域名，供后续域名规则精细匹配使用。

**整体流量决策流程：**

```text
流量进入
  ↓
是节点 IP？→ direct（防死循环）
  ↓
是端口 53？→ hijack-dns（全面接管 DNS，修正解析漂移）
  ↓
是私网 IP？→ direct（内网服务直连，零延迟）
  ↓
sniff（提取域名）
  ↓
域名匹配规则 → 走隧道优化(proxy) 或 走本地直连(direct)
  ↓
geosite/geoip 规则 → proxy 或 direct
  ↓
final: direct（兜底直连）

```

### 3.2 DNS 解析系统（应对解析异常与漂移）

DNS 配置与路由规则是一套协同工作的体系：

- **`proxy-dns`（DoT + Detour）**：通过 `8.8.8.8:853` 进行 TLS 加密查询，且设置了 `"detour": "proxy"`。这意味着解析请求本身也通过隧道发送，彻底解决在复杂广域网环境中经常遇到的 DNS 解析异常和重定向问题。
- **`local-dns`**：使用系统本地 DNS，为国内主流业务（如阿里云、腾讯云等）获取最优的 CDN 节点加速。
- **`"final": "local-dns"`**：未在规则中明确列出的域名，默认用本地解析，与路由层的直连兜底策略呼应。

**为什么 DNS 解析准确性如此重要？**

在特定的网络环境下，部分海外域名（如 GitHub API 等）的 DNS 查询可能会被重定向或分配到高延迟的节点。如果路由规则把这些错误 IP 进行直连，连接就会失败。通过 `proxy-dns` 进行加密解析，拿到准确的远端 IP，再配合路由规则分流，才能实现真正意义上的加速。

### 3.3 Inbounds 入站（双管齐下：接管本机与局域网）

- **`tun` 入站 (`strict_route: true`)**：接管本机的底层网络，强制所有流量进入虚拟网卡，彻底解决各种 IDE（如 JetBrains/Zed）和终端工具代理不生效或 DNS 泄露的问题。
- **`mixed` 入站 (`listen: "::"`)** ：监听全部地址。允许局域网内的其他设备将其作为 HTTP/SOCKS 代理网关。

### 3.4 Outbounds 出站（uTLS 流量特征混淆）

- **uTLS (Chrome Fingerprint)**：常规隧道的 TLS 指纹比较固化，容易被中间件检测到并进行 QoS 限速。开启 `uTLS` 模拟真实的 Chrome 浏览器指纹，能大幅提升复杂网络链路下的连接稳定性。

---

## 四、 常见踩坑与路由排查

### 4.1 访问海外技术网站失败（DNS 解析异常）

**现象**：`curl https://api.github.com` 报 TLS 错误，但带上本地代理端口后正常。

**排查**：执行 `dig api.github.com`，如果返回的 IP 明显存在异常或无法 ping 通。

**根因**：`route.rules` 中 DNS 接管规则（`hijack-dns`）没有在 `ip_is_private` 之前生效，导致系统发送给本地网关的 DNS 查询被当做内网流量直连放行了。

**解决**：严格按照本文配置，将 `port: 53` 的匹配规则放在私网直连规则之前。

### 4.2 连接内网微服务或数据库延迟高

**现象**：内网环境 ping 值 3ms，但 TCP 握手耗时突然增高到 300ms+。

**根因**：`sniff` 嗅探的 300ms 默认超时机制。

**解决**：将 `ip_is_private` 规则放在 `sniff` 之前，彻底避开不必要的嗅探等待。

---

## 五、 进阶用法：部署为局域网透明调度网关

如果你把这份配置运行在家里的**树莓派、闲置的 Linux 主机（如 Debian/Ubuntu 系统）或软路由**上，它可以直接化身为全团队/全家的流量调度中枢。

由于开启了 TUN 模式并监听了局域网地址，在 Linux 宿主机上，只需开启操作系统的内核 IP 转发：

```bash
# 临时开启 IP 转发
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# 永久开启 IP 转发
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

```

设置完毕后，将局域网内其他设备（如手机、开发板或测试机）的网络设置中，把 **网关 (Router/Gateway)** 和 **DNS** 均指向这台 Linux 机器的局域网 IP。所有设备将无需安装任何客户端，即可享受高度定制化的策略路由网络！

---

## 六、 验证与分流测试（必须进行）

网络调度不仅是“能连通”，更要确保“本地大流量不绕路”。配置完成后请逐一测试：

### 1. 验证国内业务直连（兜底测试）

- **测试方法**：浏览器访问 `http://www.cip.cc`。
- **结果判断**：如果显示的 IP 是你所在城市的**本地真实宽带 IP**，说明“直连兜底”和本地规则生效，国内正常业务没有浪费远程节点的带宽。

### 2. 验证 DNS 解析准确性

- **测试方法**：在终端执行 `dig github.com` 或通过测试网站检测。
- **结果判断**：返回的 IP 应当准确无误，且如果你进行 DNS 泄漏测试，结果中不应出现本地运营商的默认 DNS 服务器，说明劫持+加密解析配置完美。

---

## 七、结语

注意：使用前请确保替换配置文件中的 `UUID`、`Public_Key` 等参数。JSON 不支持注释，正式运行时务必删除代码块中的注释行。

建议：如果当前局域网不支持 IPv6，建议将 `dns.strategy` 修改为 `ipv4_only`，并精简 TUN 类型的 IPv6 地址，避免引发本地协议栈的寻址异常。

这套配置的灵魂在于“策略精准”。抛弃黑盒化的图形面板，回归最纯粹的配置代码，你会发现基于底层规则驱动网络调度的巨大魅力。通过 `final: direct` 保障了本地网络的绝对纯净，同时利用规则的优先级解决了解析异常、内网高延迟等经典难题。
