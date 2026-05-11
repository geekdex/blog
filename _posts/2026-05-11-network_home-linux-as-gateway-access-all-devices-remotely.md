---
title: 内网穿透进阶：一台家用 Linux 当网关，异地访问家里所有设备
date: 2026-05-11 15:52:00 +0800
categories: [ 网络 ]
tags: [ 异地组网, 内网穿透 ]
pin: false
---

> 你已经会用内网穿透了。但你是否遇到过这样的需求：不只想连上家里那一台服务器，还想顺手访问家里局域网里的其他设备，比如
> NAS、路由器管理页面、KVM 虚拟机，甚至整个 `192.168.0.0/24` 网段？
>
> 本文正是为此而写。

---

## 适合哪些人读

- 已经用 ZeroTier、WireGuard、FRP 或其他工具搭建好了异地互联
- 希望以家里那台 Linux 为跳板，访问家里整个局域网
- 想把配置做得优雅一点，可维护，可复用

如果还没搭建好内网穿透，可先参考这篇教程：
👉 [把服务器放在贵州深山，随时从沿海大城市访问算力](https://blog.algs.tech/posts/tools_i-placed-a-server-in-guizhou-deep-mountains-to-access-computing-power-from-coastal-big-cities-anytime.html)

---

## 场景说明

本文以如下典型场景为例：

| 角色           | 说明                                                   |
|--------------|------------------------------------------------------|
| 家里的 Linux 主机 | Debian，局域网 IP `192.168.0.100`，异地组网 IP `10.144.0.100` |
| 家里的 KVM 虚拟机  | 运行在该主机上，网段 `192.168.122.0/24`                        |
| 家里局域网        | 路由器下所有设备，网段 `192.168.0.0/24`                         |
| 办公室 / 移动端电脑  | 通过异地组网与家里主机互联                                        |

**目标：** 在办公室，通过异地组网 IP 访问家里 `192.168.122.x` 和 `192.168.0.x` 网段的所有设备，流量经由 `10.144.0.100`
（家里主机）中转。

架构示意：

```
办公室电脑
    │
    │  异地组网隧道（ZeroTier / WireGuard 等）
    ▼
10.144.0.100（家里 Linux 主机，异地组网 IP）
    ├──► 192.168.122.0/24（KVM 虚拟机）
    └──► 192.168.0.0/24（局域网所有设备）
```

---

## 系统推荐

异地互联的机器建议统一使用 Linux，推荐 **Debian**：

- 软件包兼容性好，`apt` 生态完善
- 社区活跃，遇到问题容易找到答案
- 本文所有命令均基于 Debian，其他发行版（Ubuntu、CentOS 等）理论上可用，个别路径或命令可能有差异

---

## 第一步：家里 Linux 主机开启 IP 转发

IP 转发（IP Forwarding）允许主机将收到的数据包转发到其他网络接口，是实现"网关"功能的核心。

### 临时开启（重启失效）

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 永久开启（推荐）

编辑 `/etc/sysctl.conf`，找到或添加以下行：

```ini
net.ipv4.ip_forward = 1
```

立即生效：

```bash
sudo sysctl -p
```

验证是否成功：

```bash
sysctl net.ipv4.ip_forward
# 输出应为：net.ipv4.ip_forward = 1
```

---

## 第二步：办公室 Linux 电脑配置静态路由

### 方案：使用 systemd 服务自动管理路由

直接用 `ip route add` 命令添加路由在重启后会丢失。我们用一个 systemd 服务来实现开机自动添加、关机自动清理，并且在网络未就绪时自动重试。

**配置分为两个文件：**

---

#### `/etc/static-routes.conf`（路由变量配置，日常只改这里）

```ini
# 路由网关 IP（家里主机的异地组网 IP）
GATEWAY=10.144.0.100

# 异地组网虚拟网卡名（用 ip link 查看，ZeroTier 通常以 zt 开头，WireGuard 通常为 wg0）
IFACE=ztuku5jfcs

# 目标网段，英文逗号分隔，可随意增减
ROUTES="192.168.122.0/24,192.168.0.0/24"
```

> **如何查看网卡名：**
> ```bash
> ip link show
> # ZeroTier 网卡通常以 zt 开头，WireGuard 通常为 wg0
> ```

---

#### `/etc/systemd/system/static-routes.service`（服务文件，一般不需要改动）

```ini
[Unit]
Description=Add static routes via specified gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
EnvironmentFile=/etc/static-routes.conf

# 等待网关可达，最多重试 10 次，每次间隔 3 秒（共 30 秒）
ExecStartPre=/bin/sh -c '\
  for i in 1 2 3 4 5 6 7 8 9 10; do \
    ping -c1 -W2 $GATEWAY && exit 0; \
    sleep 3; \
  done; exit 1'

ExecStart=/bin/sh -c '\
  for route in $(echo $ROUTES | tr "," " "); do \
    ip route replace $route via $GATEWAY dev $IFACE; \
  done'

ExecStop=/bin/sh -c '\
  for route in $(echo $ROUTES | tr "," " "); do \
    ip route del $route via $GATEWAY dev $IFACE 2>/dev/null || true; \
  done'

Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**几个设计细节：**

- `ExecStartPre` 先 ping 网关，确认异地组网隧道通了再添加路由，避免系统刚启动时网络未就绪导致的失败
- `ip route replace` 是幂等操作，重复执行不会报错
- `Restart=on-failure` + `RestartSec=30` 保证万一启动失败，systemd 会自动重试

---

#### 启用服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now static-routes
```

验证：

```bash
sudo systemctl status static-routes
ip route show | grep 192.168
```

日后增加新网段，只需编辑 `/etc/static-routes.conf` 的 `ROUTES` 行，然后：

```bash
sudo systemctl restart static-routes
```

---

## 第三步：办公室电脑配置静态路由

博主的办公室电脑同样是 Linux，因此本文以 Linux 的 systemd 方案作为示例，配置方式与上文完全一致，复用同一份配置文件即可。

**如果你使用的是 Windows 或 macOS**
，核心思路相同——在本机添加一条静态路由，告诉系统访问家里的网段时走异地组网的虚拟网卡。由于博主手边没有这两种环境，无法逐一验证，这里就不贴具体命令了，避免误导。建议搜索以下关键词自行配置：

- Windows：`route add` 命令，或在网络适配器高级设置中添加静态路由
- macOS：`sudo route -n add -net` 命令，或通过 LaunchDaemon 实现开机自动生效

---

## 验证全链路

配置完成后，从办公室电脑测试：

```bash
# 能否 ping 通家里 KVM 虚拟机
ping 192.168.122.x

# 能否 ping 通家里其他局域网设备
ping 192.168.0.x

# 访问家里路由器管理页面（通常是 192.168.0.1）
curl http://192.168.0.1
```

如果不通，排查顺序：

1. 异地组网是否连接正常（以 ZeroTier 为例：`zerotier-cli peers`，其他工具参考各自文档）
2. 家里主机 IP 转发是否开启：`sysctl net.ipv4.ip_forward`
3. 办公室电脑路由是否生效：`ip route show`（Linux）/ `route print`（Windows）
4. 家里主机防火墙是否放行转发流量：检查 `iptables -L FORWARD`

---

## 小结

| 步骤                  | 操作                            |
|---------------------|-------------------------------|
| 家里 Linux            | 开启 `net.ipv4.ip_forward = 1`  |
| 办公室 Linux           | 部署 `static-routes` systemd 服务 |
| 办公室 Windows / macOS | 原理相同，使用各平台路由命令自行配置            |

核心思路很简单：**让办公室电脑知道，访问家里网段要走异地组网隧道，由家里的 Linux 主机做转发中继。** 一旦理解了这个原理，不管用
ZeroTier、WireGuard 还是其他异地组网工具，配置思路完全一致，只需把网关 IP 和网卡名替换掉即可。
