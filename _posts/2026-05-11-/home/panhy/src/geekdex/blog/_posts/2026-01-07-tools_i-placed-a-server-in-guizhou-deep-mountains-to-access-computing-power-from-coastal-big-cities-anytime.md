---
title: 我在贵州深山放了台服务器，在沿海的大城市里随时调用算力
date: 2026-01-07 16:18:00 +0800
categories: [工具]
tags: [异地组网, 内网穿透]
pin: false
---

## 1. 前言：一个极客的奇思妙想

你是否曾遇到过这些痛点：

- ☕ **咖啡厅办公**，轻薄本跑个代码编译要半小时，如果能调用家里台式机的 32 核 CPU 该多好？
- 📱 **地铁通勤**，想用手机看家里 NAS 的 4K 电影，但公网 IP 太贵，内网穿透又不稳定？
- 🌏 **异地工作**，在深圳租房，老家有台闲置的高配主机吃灰，能不能远程用起来？

我们要讨论的是：

**如何让城市里的笔记本，直接调用老家主机的计算资源**  
**如何让手机，随时访问家里的所有设备和文件**  
**如果能让办公室和家里的电脑，像在同一个局域网一样互访**

等等，这些需求的本质都是一样的：**突破物理距离的限制，让你的计算资源无处不在**。

而我的方案更加极客：

我在贵州深山老家放了一台高性能主机——那里海拔高、气温低，夏天不用空调，电费便宜。父母帮忙照看，偶尔看看指示灯就行。**而我在北上广深的任何地方，都能像用本地电脑一样，随时调用这台主机的算力。**

训练 AI 模型？丢给家里的 GPU 跑。渲染视频？远程提交任务。数据处理？SSH 连上去一条命令搞定。

**这不是科幻，这就是我们今天要实现的——用 ZeroTier 打造一个跨越千里的私有云网络。**

## 2. 什么是 ZeroTier？

简单来说，ZeroTier 是一个**虚拟局域网**工具。它能让分散在世界各地的设备，就像连接在同一个路由器下一样相互通信。

传统方案的痛点：

- ❌ 动态公网IP？太贵
- ❌ 内网穿透？不稳定
- ❌ VPN？配置复杂

ZeroTier 的优势：

- ✅ 免费（个人使用）
- ✅ 配置简单（5分钟搞定）
- ✅ P2P直连（低延迟）
- ✅ 跨平台（Windows/Linux/macOS/Android/iOS）

---

## 3. 基础实战 - 连接老家和城市

### 场景描述

- **老家主机（Debian）**：贵州深山，24小时运行，电费便宜
  - 运行了多个 KVM 虚拟机用于测试和服务
- **城市笔记本（Debian）**：工作地，需要调用老家主机算力

### 目标

让城市的笔记本能够：

1. 直接 SSH 登录老家主机
2. 访问老家主机上的 KVM 虚拟机
3. 把大型计算任务丢给老家主机跑

---

### 步骤一：注册 ZeroTier 账号

1. 访问 [ZeroTier 官网](https://www.zerotier.com/)
2. 点击右上角 **Sign Up** 注册账号
3. 登录后进入控制台

---

### 步骤二：创建网络

1. 在控制台点击 **Create A Network**
2. 系统会生成一个 16 位的 Network ID，例如：`a1b2c3d4e5f6g7h8`
3. 点击网络 ID 进入配置页面
4. 记录下你的 **Network ID**（后面要用）

**关键配置：**

- **Access Control**：选择 `Private`（私有网络，需要手动授权设备）
- **IPv4 Auto-Assign**：选择一个网段，比如 `10.144.0.0/16`

---

### 步骤三：安装 ZeroTier 客户端

#### 老家主机（Debian）

```bash
# 安装 ZeroTier
curl -s https://install.zerotier.com | sudo bash

# 加入网络（替换为你的 Network ID）
sudo zerotier-cli join a1b2c3d4e5f6g7h8

# 查看状态
sudo zerotier-cli listnetworks
```

#### 城市笔记本（Debian）

```bash
# 同样的安装命令
curl -s https://install.zerotier.com | sudo bash

# 加入同一个网络
sudo zerotier-cli join a1b2c3d4e5f6g7h8
```

---

### 步骤四：授权设备

回到 ZeroTier 控制台：

1. 刷新页面，你会看到两台设备出现在 **Members** 列表
2. 勾选每台设备前面的 **Auth** 复选框（授权加入网络）
3. 可以给每台设备分配固定 IP：
   - 老家主机：`10.144.0.100`
   - 城市笔记本：`10.144.1.100`

---

### 步骤五：测试连接

在城市笔记本上：

```bash
# 查看 ZeroTier 网卡（名字类似 ztxxxxx）
ifconfig

# Ping 老家主机
ping 10.144.0.100

# SSH 登录老家主机
ssh username@10.144.0.100
```

这时候，已经成功建立了一个跨越千里的私有网络！

---

## 4. 进阶玩法 - 访问虚拟机

### 场景升级

老家主机上运行着多个 KVM 虚拟机：

- 虚拟机 1：`192.168.122.10` - 开发测试环境
- 虚拟机 2：`192.168.122.20` - 数据库服务
- 虚拟机 3：`192.168.122.30` - AI 模型训练

**问题**：虚拟机的 IP 是内网地址，城市笔记本无法直接访问。

**解决方案**：配置路由转发，让老家主机成为虚拟机的"网关"。

---

### 配置老家主机（作为网关）

#### (1) 开启 IP 转发

```bash
# 临时开启
sudo sysctl -w net.ipv4.ip_forward=1

# 永久开启
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### (2) 配置防火墙规则

```bash
# 允许 ZeroTier 和虚拟机网桥之间转发
sudo iptables -I FORWARD 1 -i ztxxxxxx -o virbr0 -j ACCEPT
sudo iptables -I FORWARD 1 -i virbr0 -o ztxxxxxx -j ACCEPT
```

> **提示**：`ztxxxxxx` 是你的 ZeroTier 网卡名，用 `ifconfig` 查看；`virbr0` 是 KVM 默认网桥。

为了保证开机自启生效，我门使用 systemd 创建服务：

```bash
sudo vim /etc/systemd/system/zt-forward.service
```

内容：

```ini
[Unit]
Description=ZeroTier Forwarding Rules
After=zerotier-one.service network-online.target
Wants=network-online.target
Requires=sys-subsystem-net-devices-ztuku5clkz.device

[Service]
Type=oneshot
RemainAfterExit=yes
Environment="ZT_IFACE=ztuku5clkz"
Environment="BR_IFACE=virbr0"

# 启动时添加规则
ExecStart=/usr/sbin/iptables -I FORWARD 1 -i ${ZT_IFACE} -o ${BR_IFACE} -j ACCEPT
ExecStart=/usr/sbin/iptables -I FORWARD 1 -i ${BR_IFACE} -o ${ZT_IFACE} -j ACCEPT

# 停止时删除规则
ExecStop=/bin/bash -c '/usr/sbin/iptables -D FORWARD -i ${ZT_IFACE} -o ${BR_IFACE} -j ACCEPT 2>/dev/null || true'
ExecStop=/bin/bash -c '/usr/sbin/iptables -D FORWARD -i ${BR_IFACE} -o ${ZT_IFACE} -j ACCEPT 2>/dev/null || true'

[Install]
WantedBy=multi-user.target
```

---

### 配置城市笔记本（添加路由）

#### 方法一：手动添加路由（临时）

```bash
# 添加到虚拟机网段的路由
sudo ip route add 192.168.122.0/24 via 10.144.0.100 dev ztxxxxxx

# 测试
ping 192.168.122.10
ssh user@192.168.122.10
```

#### 方法二：自动添加路由（开机启动）

创建 systemd 服务：

```bash
sudo nano /etc/systemd/system/zt-route.service
```

粘贴以下内容：

```ini
[Unit]
Description=Add route to home VM network via ZeroTier
After=zerotier-one.service network-online.target
Wants=network-online.target
Requires=sys-subsystem-net-devices-ztuku5clkz.device

[Service]
Type=oneshot
Environment="ZT_INTERFACE=ztuku5clkz"
ExecStart=/bin/sh -c '\
  for i in 1 2 3 4 5; do \
    ip addr show ${ZT_INTERFACE} | grep -q "inet " && break; \
    sleep 2; \
  done; \
  ip route add 192.168.122.0/24 via 10.144.0.100 dev ${ZT_INTERFACE} 2>/dev/null || true'
ExecStop=/bin/sh -c '\
  ip route del 192.168.122.0/24 2>/dev/null || true'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable zt-route.service
sudo systemctl start zt-route.service
```

---

### 验证效果

现在你可以在城市笔记本上：

```bash
# 直接访问老家的虚拟机
ssh user@192.168.122.10

# 连接虚拟机上的数据库
mysql -h 192.168.122.20 -u root -p

# 向虚拟机提交计算任务
scp large_dataset.tar.gz user@192.168.122.30:/data/
```

现在，你的老家主机现在就是一台"高性能的私有云服务器"！

---

## 5. 扩展应用 - 无限可能

### 场景 1：多地互联 - 打造分布式计算网络

假设你有三台主机：

- **老家主机**（贵州）：`10.144.1.100`，虚拟机网段 `192.168.122.0/24`
- **城市主机**（深圳）：`10.144.2.100`，虚拟机网段 `192.168.123.0/24`
- **朋友主机**（北京）：`10.144.3.100`，虚拟机网段 `192.168.124.0/24`

**配置要点：**

每台主机都：

1. 开启 IP 转发
2. 配置 iptables 转发规则
3. 添加到其他主机虚拟机网段的路由

例如，老家主机添加路由：

```bash
# 到深圳虚拟机的路由
sudo ip route add 192.168.123.0/24 via 10.144.2.100 dev ztxxxxxx

# 到北京虚拟机的路由
sudo ip route add 192.168.124.0/24 via 10.144.3.100 dev ztxxxxxx
```

**最终效果**：三地的所有虚拟机可以互相访问，形成一个跨越千里的"超级局域网"。

---

### 场景 2：办公室 + 家庭网络 - 无缝办公

**痛点**：办公室电脑上的文件、开发环境无法在家访问。

**解决方案**：

- 办公室台式机：`10.144.10.100`
- 家里笔记本：`10.144.11.100`

通过 ZeroTier 连接后：

- 在家 SSH 到办公室电脑继续写代码
- 用 Samba/NFS 共享办公室的文件
- 远程使用办公室的高性能 GPU

不过要注意的是：你要确保你公司允许你使用 ZeroTier。

---

### 场景 3：云服务器 + 本地设备 - 混合云架构

**架构设计**：

- 云服务器（腾讯云/阿里云）：`10.144.20.100`
- 家里 NAS：`10.144.21.100`
- 笔记本：`10.144.21.101`

**应用场景**：

- 云服务器作为公网入口（跳板机）
- 家里 NAS 存储私密数据（不上传公有云）
- 笔记本随时随地访问家里的文件和服务

---

### 场景 4：移动办公 - 笔记本漫游

**场景**：

- 家里台式机：`10.144.30.100`（固定运行）
- 笔记本：`10.144.30.101`（随身携带）

**无论笔记本在哪里（咖啡厅、高铁、酒店），都能通过 ZeroTier 访问家里的台式机：**

- 远程桌面
- 同步文件
- 调用家里的打印机、扫描仪

---

## 6. 实用技巧与优化

### 技巧 1：固定 IP 分配

在 ZeroTier 控制台给每台设备分配固定 IP，避免每次重启 IP 变化：

```shell
老家主机：10.144.0.100
城市笔记本：10.144.1.100
手机：10.144.1.101
```

另外，每个实际的物理局域网，只需要将一个设备加入 ZeroTier，其他设备通过该设备访问即可，减少管理复杂度。

---

### 技巧 2：监控网络状态

```bash
# 查看 ZeroTier 状态
sudo zerotier-cli status

# 查看连接的对等节点
sudo zerotier-cli listpeers

# 查看网络信息
sudo zerotier-cli listnetworks
```

---

## 7. 总结

### 解决方案回顾

通过 ZeroTier，我们实现了：

- 🏠 **老家的主机**成为你的私有云
- 💻 **城市的笔记本**随时调用远端算力
- 🌐 **多地设备**组成分布式网络

你不再受限于物理位置，计算资源可以跨越山川河流为你服务。这一切都是免费的，配置简单，维护成本几乎为零。

### ZeroTier 常用命令

```bash
# 安装
curl -s https://install.zerotier.com | sudo bash

# 加入网络
sudo zerotier-cli join <NETWORK_ID>

# 离开网络
sudo zerotier-cli leave <NETWORK_ID>

# 查看状态
sudo zerotier-cli status
sudo zerotier-cli listnetworks
sudo zerotier-cli listpeers

# 启动/停止服务
sudo systemctl start zerotier-one
sudo systemctl stop zerotier-one
sudo systemctl restart zerotier-one
```

### 结语

**注**：本文基于作者实际使用需求和经验编写，所有配置均已验证可用。如有疑问或改进建议，欢迎交流讨论！

**关键词**：ZeroTier、异地组网、虚拟局域网、KVM虚拟机、私有云、远程访问、内网穿透、分布式网络

> 本文原创首发自公众号: 【极客开发者】，禁止转载。
