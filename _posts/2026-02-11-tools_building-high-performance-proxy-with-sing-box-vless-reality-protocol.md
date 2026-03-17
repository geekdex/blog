---
title: 我用 sing-box 搭了一套极致精准的科学上网方案（全平台通用+网关化）
date: 2026-03-18 00:15:00 +0800
categories: [工具]
tags: [网络工具, 正向代理]
pin: false
---

## 一、 工具介绍与核心工作流

在尝试了诸多代理工具与繁杂的第三方 GUI 面板后，我最终将整套方案迁移到了纯粹的 **sing-box** 官方原版核心。

这套配置的设计哲学围绕以下三大核心理念展开：

1. **拒绝黑盒，精准控流**：通过详尽的规则，掌握系统里每一个数据包的走向，绝不让流量不明不白地流失。
2. **极简配置与直连兜底**：遵循极其严谨的优先级：**固定 IP > IP 范围 (CIDR) > 固定域名 > 域名后缀 > Geosite 数据库**。在国内网络环境下，我们采用**“最终使用直连（direct）做兜底”**的白名单策略。遇到打不开的外国网站再按需添加，这是最省 VPS 流量、延迟最低、最稳定的做法。
3. **全平台统一与网关化**：只使用 sing-box 官方原版核心，一份通用的 JSON 配置吃透 Linux、Windows、macOS、Android 和 iOS。在 Linux 设备上，只需开启 IP 转发，它还能瞬间化身为局域网的透明代理网关。

在动手配置之前，我们必须先理解手里的武器是什么。

### 1.1 什么是 sing-box？

sing-box 是一个比较新的网络代理/隧道工具，主要用于网络分流、代理、跨网络访问和隐私保护。它和很多人熟悉的 V2Ray、Clash、Xray 属于同一类工具，但设计更现代化。它本身不是 VPN 服务，而是一个**流量调度器 + 代理协议实现**。

换一句话说：**sing-box 就是一个可以根据规则把网络流量转发到不同代理或直连出口的高性能代理引擎。**

### 1.2 为什么选择 sing-box？

- **性能优越**：Golang 编写，内存占用极低，处理速度快。
- **配置灵活**：支持极为强大的分流规则、DNS 智能解析以及多协议兼容。
- **开源透明**：采用 MIT 协议，社区活跃，持续更新，没有闭源工具的安全隐患。

### 1.3 sing-box 的执行流（管道模型）

理解 sing-box 配置其实就是理解一个数据流管道模型。理解它最重要的三个概念是：

```text
入口 (inbound) → 路由 (route) → 出口 (outbound)
```

再加上一个 **DNS 系统** 在旁边参与决策。sing-box 的执行顺序是固定的，整体流程如下：

```text
启动
 ↓
加载配置
 ↓
启动 DNS
 ↓
启动 inbounds
 ↓
接收连接 (接收流量)
 ↓
DNS 解析 (判断这是哪个网站)
 ↓
route 匹配 (决定流量该去哪)
 ↓
选择 outbound (直连还是走代理发出)
 ↓
发送
```

### 1.4 了解 VLESS 协议

我们要搭的底层通道使用的是 VLESS 协议。VMess 是一种网络代理协议，最早由 V2Ray 项目设计，用于科学上网和隐私保护。而 VLESS 则是 VMess 的改进版本，它的设计目标是：**更轻量、更灵活、减少协议复杂度，同时降低被特征识别的概率。**

> 一句话总结：VLESS = 去掉验证复杂机制的轻量级代理协议。

它本质上也是客户端 ↔ 服务器之间的加密代理通信协议，采用代理转发模式：

```text
客户端 (本机 sing-box)
        │
   VLESS 加密通信
        │
服务器 (境外 VPS 的 VLESS 服务端)
        │
     真实互联网 (如 Google、X)
```

**完整流程**：

1. 客户端请求某个网站（比如 Google）。
2. 请求数据通过 VLESS 进行封装。
3. 通过 TLS / Reality / XTLS 等方式加密传输，伪装成正常流量。
4. 服务器解密后代为访问目标网站。
5. 服务器把响应数据再加密返回给客户端。

### 1.5 我们的实现目标

实际上 sing-box 还支持多种协议，但接下来我们将基于上述介绍的 **VLESS 协议** 实现最安全的科学上网。下文的全套流程使用 `sing-box 1.13.3` 成功验证，在使用过程中，请根据实际环境与需求进行替换修改。

---

## 二、 服务端配置：构建隐形通道

服务端我们采用目前抗封锁能力最顶级的组合：**VLESS + XTLS-Vision + Reality**。通过“借用” `www.bing.com` 的身份，彻底消除 TLS 代理特征。

### 2.1 安装与环境准备

在你的 VPS 服务器上执行官方一键安装脚本：

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

接着，我们需要生成专属的连接凭证。使用 `sing-box generate` 命令生成 `uuid`、`reality-keypair`（私钥/公钥）以及 `short_id`：

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
        "timestamp": true
    },
    "inbounds":[
        {
            "type": "vless",
            "listen": "::",
            "listen_port": 443, 
            "users":[
                {
                    "uuid": "YOUR_UUID", // 填入你生成的 UUID
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
                    "private_key": "YOUR_PRIVATE_KEY", // 填入你生成的私钥
                    "short_id":["YOUR_SHORT_ID"] // 填入你生成的 Short ID
                }
            }
        }
    ],
    "outbounds":[
        {
            "type": "direct",
            "tag": "direct"
        }
    ]
}
```

**精细解析：**

- **XTLS-Vision**：VLESS 协议的精髓，能识别出加密流量里的数据包长度特征并进行“伪装填充”，专门对付防火墙对 TLS 代理的精准识别。
- **Reality**：它直接转发了 `www.bing.com` 的握手响应。对于外部探测者来说，你的服务器就是一个完美的 Bing 镜像，没有任何代理迹象，实现了彻底的隐蔽。

---

## 三、 客户端配置：全平台通用的智能大脑

客户端配置是这套系统的核心大脑。以下是完整的示例配置，**你可以直接将它保存在任何平台的 sing-box 客户端中使用**（别忘了替换 `server` IP、`uuid`、`public_key` 等参数）：

```json
{
    "log": {
        "disabled": false,
        "level": "warn",
        "timestamp": true
    },
    "dns": {
        "servers":[
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
        "rules":[
            {
                "domain": ["local.dev", "internal.corp"],
                "server": "local-dns"
            },
            {
                "domain_suffix":[
                    "bing.com",
                    "microsoft.com",
                    "msftconnecttest.com",
                    "msftncsi.com"
                ],
                "server": "local-dns"
            },
            {
                "rule_set": "geosite-cn",
                "server": "local-dns"
            },
            {
                "rule_set": [
                    "geosite-google",
                    "geosite-youtube",
                    "geosite-twitter",
                    "geosite-facebook",
                    "geosite-instagram",
                    "geosite-telegram",
                    "geosite-netflix",
                    "geosite-github",
                    "geosite-openai",
                    "geosite-apple"
                ],
                "server": "proxy-dns"
            }
        ],
        "final": "local-dns",
        "independent_cache": true,
        "strategy": "prefer_ipv4"
    },
    "inbounds":[
        {
            "type": "tun",
            "tag": "tun-in",
            "address":[
                "172.19.0.1/30",
                "fdfe:dcba:9876::1/126"
            ],
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
            "listen": "0.0.0.0",
            "listen_port": 2334
        }
    ],
    "outbounds":[
        {
            "type": "vless",
            "tag": "proxy",
            "server": "YOUR_VPS_IP", // 替换为你 VPS 的真实 IP
            "server_port": 443,
            "uuid": "YOUR_UUID", // 你的 UUID
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
                    "public_key": "YOUR_PUBLIC_KEY", // 你的 Reality 公钥
                    "short_id": "YOUR_SHORT_ID" // 你的 Short ID
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
        "rule_set":[
            {
                "tag": "geosite-cn",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs",
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
                "tag": "geosite-youtube",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-youtube.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geosite-twitter",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-twitter.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geosite-facebook",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-facebook.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geosite-instagram",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-instagram.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geosite-telegram",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-telegram.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geosite-netflix",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-netflix.srs",
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
                "tag": "geosite-apple",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-apple.srs",
                "download_detour": "proxy"
            },
            {
                "tag": "geoip-cn",
                "type": "remote",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs",
                "download_detour": "proxy"
            }
        ],
        "rules":[
            { "action": "sniff" },
            { "protocol": "dns", "action": "hijack-dns" },
            
            // 优先级 1: 局域网与私有 IP 绝对直连
            { "ip_is_private": true, "outbound": "direct" },
            
            // 优先级 2: 固定 IP 精准控制
            { "ip_cidr":["1.2.3.4/32", "5.6.7.8/32"], "outbound": "direct" },
            
            // 优先级 3: IP 范围 (CIDR) 控制
            { "ip_cidr":["103.24.0.0/16", "45.128.100.0/24"], "outbound": "direct" },
            
            // 优先级 4: 固定域名匹配
            { "domain": ["local.dev", "internal.corp"], "outbound": "direct" },
            
            // 优先级 5: 域名后缀匹配
            {
                "domain_suffix":["bing.com", "microsoft.com", "msftconnecttest.com", "msftncsi.com"],
                "outbound": "direct"
            },
            
            // 优先级 6: Geosite / GeoIP 数据库匹配
            { "rule_set": "geoip-cn", "outbound": "direct" },
            { "rule_set": "geosite-cn", "outbound": "direct" },
            { "rule_set": "geosite-google", "outbound": "proxy" },
            { "rule_set": "geosite-youtube", "outbound": "proxy" },
            { "rule_set": "geosite-twitter", "outbound": "proxy" },
            { "rule_set": "geosite-facebook", "outbound": "proxy" },
            { "rule_set": "geosite-instagram", "outbound": "proxy" },
            { "rule_set": "geosite-telegram", "outbound": "proxy" },
            { "rule_set": "geosite-netflix", "outbound": "proxy" },
            { "rule_set": "geosite-github", "outbound": "proxy" },
            { "rule_set": "geosite-openai", "outbound": "proxy" },
            { "rule_set": "geosite-apple", "outbound": "proxy" }
        ],
        // 最终兜底：国内环境默认全直连
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

### 3.1 路由匹配策略（核心：直连兜底与清晰的优先级）

仔细看 `route.rules` 的配置，sing-box 的执行流是**自上而下匹配**的。我构建了一个极其严谨的分流层级：

- **优先级分层**：固定 IP > IP 范围 (CIDR) > 固定域名 > 域名后缀 > 数据库 (Geosite/GeoIP)。
- **直连兜底 (`"final": "direct"`)**：因为我们身处国内，绝大部分未知的流量（如冷门国内 App、下载工具等）本质上都是国内流量。**使用直连作为最终兜底，是性能最高、最省 VPS 流量的做法。**
- **按需添加**：如果遇到某个外国网站无法访问，只需在“域名后缀”规则中加上该域名，并指向 `proxy` 即可，真正做到指哪打哪。
- **二进制规则 (`.srs`)**：放弃了臃肿的文本规则，采用 sing-box 专用的 `.srs` 远程二进制规则集，解析速度极快，大幅降低了内存占用。

### 3.2 Inbounds 入站（双管齐下：接管本机与局域网）

- **`tun` 入站 (`strict_route: true`)**：接管本机的底层网络。分配了 IPv4/IPv6 双栈私有地址，强制所有流量进入 TUN 虚拟网卡，防止漏网之鱼，彻底解决各类应用的代理绕过和 DNS 泄露问题。
- **`mixed` 入站 (`listen: "0.0.0.0"`)**：监听了全 0 地址。这意味着不仅本机可以使用，局域网内的其他设备也可以通过这台机器的内网 IP 发送 HTTP/SOCKS 代理请求。

### 3.3 DNS 解析系统（DoT 无污染防劫持）

- **DoT + Detour**：这是目前最安全的 DNS 方案。通过 `8.8.8.8:853` 进行 TLS 加密查询，且设置了 `"detour": "proxy"`。这意味着不仅查询是加密的，**连查询本身都被塞进了代理隧道里发往海外**，国内运营商永远无法知道你在解析什么域名。
- **`"final": "local-dns"`**：与直连兜底策略呼应。不在规则内的域名默认用本地系统 DNS 解析，获取国内最优的 CDN 节点，保证日常国内互联网访问的低延迟。

### 3.4 Outbounds 出站（uTLS 指纹隐蔽）

- **uTLS (Chrome Fingerprint)**：正常的代理工具 TLS 指纹非常独特，极易被防火墙主动探测识别。开启 `uTLS` 并指定 `chrome` 后，你的底层连接特征看起来就像是一个真实的 Chrome 浏览器在正常访问 Bing 官网，完美隐身。

---

## 四、 进阶用法：将 Linux 变为局域网透明网关

这套配置的一个巨大优势在于其强大的跨端和网关通用性。如果你把这份配置运行在家里的**树莓派、闲置 Linux 虚拟机或软路由**上，它可以直接化身为全家的代理中枢。

由于我们在配置中开启了 TUN 模式并监听了局域网地址，在 Linux 系统上，你只需要开启操作系统的 IP 转发功能：

```bash
# 临时开启 IP 转发
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# 永久开启 IP 转发，修改配置并使其生效
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

设置完毕后，将局域网内其他设备（如 Apple TV、Switch、手机等）的网络设置打开，把 **网关 (Router/Gateway)** 和 **DNS** 修改为这台运行着 sing-box 的 Linux 机器的局域网 IP。

奇迹发生了：这些设备不需要安装任何客户端，就能立刻享受透明代理，且所有的流量走向都会严格遵循你在 sing-box 设定的底层优先级路由流转！

---

## 五、 验证与分流测试（必须进行）

科学上网不仅是“能打开外网”，更要确保“国内流量不绕路”。配置完成后请逐一测试：

### 1. 验证国内网站直连（兜底测试）

- **测试方法**：浏览器访问 `http://www.cip.cc`。

- **结果判断**：如果显示的 IP 是你宽带所在的**国内真实城市 IP**，说明“直连兜底”和 `geosite-cn` 规则生效，国内大流量没有浪费你宝贵的 VPS 代理资源。

### 2. 验证精确分流与按需添加

- **测试方法**：在配置的 `route.rules` 的域名后缀和 `dns.rules` 中加上 `.gs` 指向 `proxy`。

- **结果验证**：访问 `https://ip.gs`，如果显示的地址是你的 **VPS 服务器 IP**，说明你的按需精准控流完美成功。

### 3. 验证 DNS 是否泄露

- **测试方法**：在配置的 `route.rules` 的精准域名和 `dns.rules` 中加上 `dnsleaktest.com` 指向 `proxy`，访问 `https://dnsleaktest.com` 并进行标准测试。

- **结果判断**：如果你测试结果里只看到了 Google (8.8.8.8) 的 DNS 服务器，没有出现任何中国国内运营商的 DNS，说明劫持+隧道解析完美，你的上网轨迹已实现绝对隐匿。

---

## 六、结语

注意：请确保将配置文件中的 `UUID`、`Public_Key`、`Private_Key`、`Short_id` 以及 `Server_IP` 替换为你实际生成的值，同时示例配置的注释正式使用的时候要删除，避免破坏 JSON 的语法规则。

这套配置的灵魂在于‘精准打击’。抛弃复杂的图形面板，回归最纯粹的 JSON 配置文件，你会发现 sing-box 强大的控制力。通过 `final: direct` 确保了国内流量的绝对纯净，而通过 `geosite-google` 和手动维护的规则集，实现了对海外核心服务的精确接管。这是一种‘保守但极其稳健’的配置方案，是一套值得长期使用并根据个人需求不断打磨的终极方案。

另外，更多的 `rule-set` 数据库可以从以下地址查看，可根据需求来配置，rule-set 分支浏览文件列表：

- geosite 列表： <https://github.com/SagerNet/sing-geosite/tree/rule-set>
- geoip 列表： <https://github.com/SagerNet/sing-geoip/tree/rule-set>

所有 .srs 文件就是可用的 tag 名，文件名去掉 .srs 后缀就是 tag 值。

