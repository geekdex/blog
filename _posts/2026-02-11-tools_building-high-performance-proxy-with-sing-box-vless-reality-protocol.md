---
title: 使用 sing-box 自建高性能科学上网的快速方案 (2026 实战版)
date: 2026-02-11 12:26:00 +0800
categories: [工具]
tags: [网络工具, 正向代理]
pin: false
---

## 一、工具介绍

### 1.0 什么是 sing-box ？

sing-box 是一个比较新的 网络代理/隧道工具，主要用于 网络分流、代理、跨网络访问、隐私保护。它和很多人熟悉的 V2Ray、Clash、Xray 属于同一类工具，但设计更现代化。它本身不是 VPN 服务，而是一个 流量调度器 + 代理协议实现。

换一句话说：sing-box 就是一个可以根据规则把网络流量转发到不同代理或直连出口的高性能代理引擎。

### 1.1 为什么选择 sing-Box？

- **性能优越**：Golang 编写，内存占用低，处理速度快
- **配置灵活**：支持分流规则、DNS 智能解析、多协议兼容
- **开源透明**：MIT 协议，社区活跃，持续更新

### 1.3 快速开始

#### 1.3.1 了解 sing-box 的执行流程

我们简单来说一下 sing-box 的执行流程，sing-box 的配置其实是一个数据流管道模型。理解它最重要的三个概念是：

```shell
入口 → 路由 → 出口
```

也就是：

```shell
inbound  →  route  →  outbound
```

再加上一个 DNS 系统 在旁边参与决策。sing-box 的执行顺序是固定的，整体流程：

```shell
启动
 ↓
加载配置
 ↓
启动 DNS
 ↓
启动 inbounds
 ↓
接收连接(接收流量)
 ↓
DNS解析
 ↓
route匹配
 ↓
选择outbound
 ↓
发送
```

#### 1.3.2 了解 vless 协议

VMess 是一种网络代理协议，最早由 V2Ray 项目设计，用于科学上网、代理转发、流量混淆和隐私保护。它本质上是 客户端和服务器之间的一种通信协议，类似 HTTP、TLS 这种协议，但专门为代理场景设计。VLESS 是对 VMess 的改进版本，同样出自 V2Ray 体系，后来也被 sing-box 等客户端支持。它的设计目标是：更轻量、更灵活、减少协议复杂度，同时降低被特征识别的概率。

简单说一句话：
> VLESS = 去掉验证复杂机制的轻量级代理协议。

它本质上也是客户端 ↔ 服务器之间的加密代理通信协议，VLESS 依然采用代理转发模式：

```shell
客户端 (sing-box / Xray / Clash)
        │
   VLESS 加密通信
        │
服务器 (VLESS 服务端)
        │
     真实互联网
```

流程：

- 1.客户端请求某个网站（比如 Google）
- 2.请求数据通过 VLESS 进行封装
- 3.通过 TLS / Reality / XTLS 等方式加密传输
- 4.服务器解密后访问目标网站
- 5.服务器把响应数据再加密返回

#### 1.3.3 实现目标

实际上 sing-box 还支持多种协议，可以参考官方文档，接下来我们就基于上述介绍的 vless 协议实现科学上网。下文流程使用 `sing-box 1.13.1` 成功验证，在使用过程，请根据实际环境与需求进行修改。

---

## 二、服务端配置

### 2.1 安装 sing-Box

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

### 2.2 生成密钥

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

### 2.3 创建配置文件

编辑 `/etc/sing-box/config.json`：

```json
{
  "log": {
    "disabled": true,
    "level": "warn",
    "output": "/var/log/sing-box.log",
    "timestamp": true
  },
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
  }],
  "outbounds": [{
    "type": "direct"
  }]
}
```

上面的配置只有一个 outbound，它会自动成为默认出口。上述配置将实现这样一台服务器：

- 👉 运行在云服务器上的 sing-box
- 👉 作为 VLESS + Reality 代理服务端
- 👉 监听 443
- 👉 使用 bing 作为伪装域名

核心功能：接收客户端的加密代理连接，用 Reality 伪装成访问 `www.bing.com`，代理流量通过 VLESS 转发。

### 2.4 启动服务

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
- 伪装域名 `www.bing.com` 可替换为其他支持 TLS 1.3 的网站，但要确定该网站能正常访问，也就是：TLS 握手成功、证书校验成功、HTTP 层响应成功

---

## 三、客户端配置

### 3.1 安装 sing-box

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

### 3.2 完整配置文件解析

下面是一份完整可用的客户端配置，路径为 `/etc/sing-box/config.json`。我们逐块拆解每一部分的含义。

```json
{
    "log": {
        "level": "info"
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
                "tag": "local",
                "type": "local"
            }
        ],
        "rules": [
            {
                "domain_suffix": [
                    "google.com",
                    "youtube.com"
                ],
                "server": "proxy-dns"
            }
        ],
        "final": "local",
        "independent_cache": true
    },
    "inbounds": [
        {
            "type": "tun",
            "tag": "tun-in",
            "address": ["172.19.0.1/30", "fdfe:dcba:9876::1/126"],
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
    "outbounds": [
        {
            "type": "vless",
            "tag": "proxy",
            "server": "YOUR_SERVER_IP",
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
        }
    ],
    "route": {
        "default_domain_resolver": "local",
        "rules": [
            {
                "action": "sniff"
            },
            {
                "protocol": "dns",
                "action": "hijack-dns"
            },
            {
                "ip_is_private": true,
                "outbound": "direct"
            },
            {
                "domain_suffix": [
                    "google.com",
                    "youtube.com"
                ],
                "outbound": "proxy"
            }
        ],
        "final": "direct",
        "auto_detect_interface": true
    }
}
```

---

### 3.3 配置逐块详解

#### 3.3.1 log（日志）

```json
"log": {
    "level": "info"
}
```

控制 sing-box 输出日志的详细程度。`info` 级别会记录连接建立、路由匹配等基本信息，便于排查问题。可选值从详到简依次为：`trace` → `debug` → `info` → `warn` → `error`。生产环境可改为 `warn` 减少输出。

---

#### 3.3.2 dns（DNS 解析系统）

DNS 是整个分流的核心。sing-box 需要自己接管 DNS 解析，才能在域名层面做出路由决策（走代理还是直连）。

```json
"servers": [
    {
        "tag": "proxy-dns",
        "type": "tls",
        "server": "8.8.8.8",
        "detour": "proxy"
    },
    {
        "tag": "local",
        "type": "local"
    }
]
```

这里定义了两个 DNS 服务器：

**proxy-dns**：类型为 `tls`，即 DNS over TLS（DoT），查询请求走加密通道发往 Google 的 `8.8.8.8`。`"detour": "proxy"` 表示这个 DNS 查询本身也要走代理出口，防止被运营商拦截或污染。这个服务器专门用来解析需要翻墙的域名（如 google.com），通过代理查询可以拿到未被污染的真实 IP。

**local**：类型为 `local`，直接使用系统本地 DNS（即 `/etc/resolv.conf` 中配置的 DNS，一般是路由器或运营商的 DNS）。用于解析国内域名，速度快、无污染风险。

```json
"rules": [
    {
        "domain_suffix": ["google.com", "youtube.com"],
        "server": "proxy-dns"
    }
],
"final": "local"
```

DNS 规则：凡是域名后缀匹配 `google.com` 或 `youtube.com` 的查询，交给 `proxy-dns` 解析；其余所有域名走 `local` 本地 DNS 解析（`final` 是默认兜底）。

```json
"independent_cache": true
```

让每个 DNS 服务器独立维护自己的缓存，避免不同 DNS 服务器之间缓存污染。

---

#### 3.3.3 inbounds（入站）

入站定义了 sing-box 如何接收本机的流量。这里配置了两种入站方式：

**TUN 模式**：

```json
{
    "type": "tun",
    "tag": "tun-in",
    "address": ["172.19.0.1/30", "fdfe:dcba:9876::1/126"],
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
}
```

TUN 是一种虚拟网卡技术。sing-box 在系统里创建一张名为 `tun0` 的虚拟网卡，所有网络流量先经过这张虚拟网卡，再由 sing-box 决定如何处理。

- `"address": ["172.19.0.1/30", "fdfe:dcba:9876::1/126"]`：虚拟网卡同时分配 IPv4 和 IPv6 地址，均为私有地址段，仅在本机内部使用。IPv4 地址 `172.19.0.1/30` 用于接管 IPv4 流量；IPv6 地址 `fdfe:dcba:9876::1/126` 用于接管 IPv6 流量。在安卓等移动平台上，如果只配 IPv4 地址，系统发出的 IPv6 DNS 查询（`AAAA` 记录）可能绕过 TUN 网卡直接发出，造成 DNS 泄漏，因此推荐始终同时配置两个地址。Linux / Windows / macOS / Android 均支持此写法。在 Android 上推荐同时分配一个 IPv6 地址，写成数组形式：`"address": ["172.19.0.1/30", "fdfe:dcba:9876::1/126"]`，可以让 TUN 网卡同时处理 IPv4 和 IPv6 流量，避免 IPv6 流量绕过 TUN 直接出去导致 DNS 泄漏。
- `"mtu": 9000`：最大传输单元，设大一点可以减少分包，提升性能。
- `"auto_route": true`：自动向系统路由表添加规则，让流量默认流入 TUN 网卡，实现全局代理。
- `"strict_route": true`：更严格的路由模式，防止流量绕过 TUN 网卡直接出去，确保所有流量都被 sing-box 接管。
- `"platform.http_proxy"`：同时开启系统 HTTP 代理，指向本机 2334 端口（即下面的 mixed-in），方便一些不走 TUN 的程序（如某些 curl、wget）也能走代理。

**Mixed 混合代理**：

```json
{
    "type": "mixed",
    "tag": "mixed-in",
    "listen": "0.0.0.0",
    "listen_port": 2334
}
```

在本机 2334 端口同时监听 HTTP 和 SOCKS5 代理。`"listen": "0.0.0.0"` 表示监听所有网卡，局域网内其他设备也可以将代理指向这台机器的 2334 端口使用。可以直接用 `curl -x socks5h://127.0.0.1:2334 https://www.google.com` 测试。

---

#### 3.3.4 outbounds（出站）

出站定义了流量最终如何发出去，有几个出站就有几条出路。

**VLESS 代理出站**：

```json
{
    "type": "vless",
    "tag": "proxy",
    "server": "YOUR_SERVER_IP",
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
}
```

- `"type": "vless"`：使用 VLESS 协议连接服务器。
- `"server"` / `"server_port"`：服务端的 IP 和端口。
- `"uuid"`：身份验证凭据，客户端和服务端必须一致，类似密码。
- `"flow": "xtls-rprx-vision"`：启用 XTLS Vision 流控，这是一种更高效的传输方式，可以减少一层加密开销，同时让流量特征更接近正常 TLS 流量。
- `"tls.server_name": "www.bing.com"`：TLS 握手时使用的 SNI（服务器名称指示），用于伪装。防火墙看到的是客户端在访问 bing.com，而不是代理服务器。
- `"utls.fingerprint": "chrome"`：伪造 TLS 指纹为 Chrome 浏览器。正常的代理工具有独特的 TLS 指纹，容易被识别；伪造成 Chrome 后，流量特征与普通浏览器访问无异。
- `"reality"`：Reality 是一种新型伪装技术，不需要自己准备 TLS 证书。它把客户端的握手流量"借用"到真实存在的目标网站（这里是 bing.com）上，握手过程中如果没有正确的 `public_key` 和 `short_id`，防火墙只能看到一个正常的 TLS 连接，无法判断是代理流量。

**直连出站**：

```json
{
    "type": "direct",
    "tag": "direct"
}
```

直接通过本机网络发出流量，不经过代理。用于国内流量、局域网流量等不需要代理的场景。

---

#### 3.3.5 route（路由规则）

路由是决策中心，决定每一条连接最终走哪个 outbound。规则从上到下依次匹配，第一条命中即生效，不再继续往下。

```json
"default_domain_resolver": "local"
```

当路由需要对某个域名做 IP 解析时（比如判断是否为私有 IP），默认使用 `local` DNS 服务器。

```json
{ "action": "sniff" }
```

对所有进入的连接做协议嗅探。sing-box 会检查数据包内容，尝试从中识别出真实的目标域名（例如从 TLS 握手的 SNI 字段读出 `google.com`）。这一步非常关键：没有嗅探，sing-box 只能看到一个 IP 地址；有了嗅探，才能拿到域名，后续的 `domain_suffix` 规则才能生效。

```json
{ "protocol": "dns", "action": "hijack-dns" }
```

劫持 DNS 流量。当系统发出 DNS 查询时（目标端口 53），sing-box 拦截这些请求，交由自己内部的 DNS 系统处理，而不是让它直接发到系统配置的 DNS 服务器。这样 sing-box 才能对不同域名使用不同的 DNS 服务器（比如 google.com 走 proxy-dns，其他走 local），实现 DNS 分流。没有这条规则，所有 DNS 查询都会绕过 sing-box 直接发出去，分流就形同虚设。

```json
{ "ip_is_private": true, "outbound": "direct" }
```

目标 IP 是私有地址（如 `192.168.x.x`、`10.x.x.x`、`172.16-31.x.x`）的流量，直接走直连，不经过代理。局域网通信不应该绕路到代理服务器。

```json
{
    "domain_suffix": ["google.com", "youtube.com"],
    "outbound": "proxy"
}
```

域名后缀匹配 `google.com` 或 `youtube.com` 的连接，走代理出站。这里依赖前面 `sniff` 步骤嗅探出的域名。`domain_suffix` 的匹配逻辑是：目标域名等于 `google.com` 本身，或者是它的子域名（如 `www.google.com`、`mail.google.com`）。

```json
"final": "direct"
```

所有规则都没有命中时，默认走直连。也就是说，没有在代理列表里的域名和 IP，都直接访问，不走代理。

```json
"auto_detect_interface": true
```

自动检测系统的默认出口网卡（如 `enp3s0`、`eth0`），让直连流量从正确的网卡出去，而不是误走 TUN 网卡形成回环。

---

### 3.4 启动与测试

编辑好配置文件后，将 `YOUR_SERVER_IP`、`YOUR_UUID`、`YOUR_PUBLIC_KEY`、`YOUR_SHORT_ID` 替换为服务端生成的实际值，然后执行：

```bash
# 验证配置语法
sudo sing-box check -c /etc/sing-box/config.json

# 重启服务
sudo systemctl restart sing-box

# 测试（不加代理参数，依赖 TUN 自动接管）
curl "https://www.google.com"

# 也可以通过 mixed 端口测试
curl -x socks5h://127.0.0.1:2334 "https://www.google.com"
```

如果两种方式都能正常返回，说明 TUN 全局代理和手动代理均已生效。

### 3.5 添加更多代理域名

只需在 `dns.rules` 和 `route.rules` 中同时添加对应的 `domain_suffix` 即可：

```json
// dns.rules 中
{
    "domain_suffix": ["google.com", "youtube.com", "twitter.com", "github.com"],
    "server": "proxy-dns"
}

// route.rules 中
{
    "domain_suffix": ["google.com", "youtube.com", "twitter.com", "github.com"],
    "outbound": "proxy"
}
```

两处需要保持同步，否则会出现 DNS 走代理解析、但路由却走直连的情况（或反之）。
