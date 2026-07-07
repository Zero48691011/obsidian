# VPN 详解：原理与配置

---

## 一、什么是 VPN？

### 1.1 官方定义

> **VPN（Virtual Private Network，虚拟专用网络）** 是一种在公共网络（互联网）上建立**加密隧道**的技术，让远程设备可以像在本地局域网一样安全地访问私有网络资源。

### 1.2 一句话通俗解释

> VPN 就是一条**加密的秘密通道**——你在咖啡厅的公共 WiFi 上连公司内网，别人看到的是加密乱码，只有你和公司服务器知道里面是什么。

### 1.3 类比

```
没有 VPN：
  你在星巴克（公共网络）大声喊：「老王！帮我查一下客户表！」
  → 所有人都听见了，还能记下来

有了 VPN：
  你拿出一个加密对讲机（VPN 隧道），只有老王有解密钥匙
  你喊的内容被加密成乱码，别人听到的只是「滋滋滋」
```

---

## 二、VPN 的原理

### 2.1 核心：加密隧道

```
原始数据包                    加密隧道                    原始数据包
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│ 客户端        │          │   互联网       │          │ VPN 服务器    │
│              │   加密    │              │   解密    │              │
│ 192.168.1.5  │─────────▶│ 乱码乱码乱码   │─────────▶│ 10.0.0.5    │
│              │          │              │          │              │
│ 明文请求      │          │ 别人看到的是    │          │ 还原为明文     │
│              │          │ 加密后的乱码    │          │ 转发给内网     │
└──────────────┘          └──────────────┘          └──────────────┘
```

### 2.2 数据包封装过程

```
1. 原始数据包：
   ┌──────────┬───────────┬────────────┐
   │ 源 IP     │ 目标 IP     │ 数据        │
   │ 192.168.1.5│ 10.0.0.100│ GET /api   │
   └──────────┴───────────┴────────────┘

2. VPN 加密后（外层包装）：
   ┌──────────┬───────────┬────────────────────────────────┐
   │ 源 IP     │ 目标 IP     │ 加密后的整个内层数据包              │
   │ 咖啡厅 WiFi │ VPN 服务器   │ (原始包被加密，变成乱码)           │
   └──────────┴───────────┴────────────────────────────────┘

3. 到达 VPN 服务器后：
   → 剥掉外层 → 解密 → 还原原始数据包 → 转发给 10.0.0.100
```

---

## 三、常见 VPN 协议对比

| 协议 | 速度 | 安全性 | 配置难度 | 适用场景 |
|------|------|--------|---------|---------|
| **WireGuard** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **首选推荐**，现代极简协议 |
| **OpenVPN** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 兼容性最好，功能最全 |
| **IPsec/IKEv2** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | 移动端稳定，企业级方案 |
| **PPTP** | ⭐⭐⭐⭐⭐ | ⭐（已破解） | ⭐⭐⭐⭐⭐ | ❌ 不推荐，不安全 |
| **L2TP/IPsec** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 较老的企业方案 |

### 推荐原则

```
个人/小团队 → WireGuard（配置简单，性能最好）
企业环境    → OpenVPN（功能全面，认证方式丰富）
移动设备    → IKEv2（断线重连快，WiFi 切 4G 不掉线）
```

---

## 四、WireGuard 配置（推荐）

### 4.1 WireGuard 是什么

> WireGuard 是一个极简、高性能的 VPN 协议，代码仅 ~4000 行（OpenVPN 的 1/100），已并入 Linux 内核 5.6+。

**优势**：
- 配置极其简单（公钥 + 私钥 + 几行 IP 就行）
- 性能极高（内核态运行，比 OpenVPN 快 3-4 倍）
- 漫游支持（IP 变化自动重连，无需重启）
- 加密算法先进（ChaCha20 + Poly1305）

### 4.2 安装

```bash
# Ubuntu / Debian
sudo apt install wireguard

# CentOS / RHEL 8+
sudo dnf install wireguard-tools

# macOS
brew install wireguard-tools

# 验证安装
wg --version
```

### 4.3 生成密钥对

```bash
# 服务器端
wg genkey | tee server_private.key | wg pubkey > server_public.key

# 客户端
wg genkey | tee client_private.key | wg pubkey > client_public.key

# 查看生成的密钥
cat server_private.key   # 服务端私钥（保密！）
cat server_public.key    # 服务端公钥（给客户端）
cat client_private.key   # 客户端私钥（保密！）
cat client_public.key    # 客户端公钥（给服务端）
```

### 4.4 服务端配置

```bash
# 创建配置文件
sudo vim /etc/wireguard/wg0.conf
```

```ini
[Interface]
# 服务端 VPN 内网地址
Address = 10.0.0.1/24
# 监听端口
ListenPort = 51820
# 服务端私钥
PrivateKey = <服务端私钥>

# 开启转发（让客户端能访问服务端所在网络）
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# 客户端 1
[Peer]
# 客户端公钥
PublicKey = <客户端1公钥>
# 允许的客户端 IP（可以限制客户端只能访问哪些 IP）
AllowedIPs = 10.0.0.2/32

# 客户端 2
[Peer]
PublicKey = <客户端2公钥>
AllowedIPs = 10.0.0.3/32
```

```bash
# 开启 IP 转发
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf

# 启动 WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# 查看状态
sudo wg show
```

### 4.5 客户端配置

```ini
# /etc/wireguard/wg0.conf（Linux）或导入到 WireGuard App

[Interface]
# 客户端 VPN 内网地址
Address = 10.0.0.2/24
# 客户端私钥
PrivateKey = <客户端私钥>
# DNS（可选，通过 VPN 隧道解析）
DNS = 8.8.8.8

[Peer]
# 服务端公钥
PublicKey = <服务端公钥>
# 服务端公网 IP 和端口
Endpoint = <服务端公网IP>:51820
# 哪些流量走 VPN 隧道
# 0.0.0.0/0  = 全部流量走 VPN（全局代理）
# 10.0.0.0/24 = 只有 VPN 内网流量走隧道（分流）
AllowedIPs = 10.0.0.0/24
# 保持连接（NAT 穿透）
PersistentKeepalive = 25
```

### 4.6 流量模式选择

```ini
# 模式 1：分流（推荐）—— 只有内网流量走 VPN
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24

# 模式 2：全局代理 —— 所有流量都走 VPN
AllowedIPs = 0.0.0.0/0

# 模式 3：只代理特定 IP
AllowedIPs = 10.0.0.100/32
```

### 4.7 常用管理命令

```bash
# 启动/停止
sudo wg-quick up wg0
sudo wg-quick down wg0

# 查看状态
sudo wg show                    # 全部信息
sudo wg show wg0                # 指定接口
sudo wg show wg0 latest-handshakes  # 最近握手时间

# systemd 管理
sudo systemctl start wg-quick@wg0
sudo systemctl stop wg-quick@wg0
sudo systemctl restart wg-quick@wg0
sudo systemctl status wg-quick@wg0

# 动态添加客户端（无需重启）
sudo wg set wg0 peer <客户端公钥> allowed-ips 10.0.0.4/32
sudo wg-quick save wg0    # 保存到配置文件
```

---

## 五、OpenVPN 配置

### 5.1 服务端安装

```bash
# Ubuntu
sudo apt install openvpn easy-rsa

# 初始化 PKI（公钥基础设施）
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass           # 生成 CA 证书
./easyrsa gen-req server nopass     # 生成服务端证书请求
./easyrsa sign-req server server    # 签发服务端证书
./easyrsa gen-dh                    # 生成 DH 参数
./easyrsa gen-req client1 nopass    # 生成客户端证书请求
./easyrsa sign-req client client1   # 签发客户端证书
```

### 5.2 服务端配置文件

```conf
# /etc/openvpn/server.conf
port 1194
proto udp
dev tun

# 证书路径
ca   /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key  /etc/openvpn/server.key
dh   /etc/openvpn/dh.pem

# VPN 内网地址段
server 10.8.0.0 255.255.255.0

# 推送路由（让客户端能访问服务端内网）
push "route 192.168.1.0 255.255.255.0"

# 推送 DNS
push "dhcp-option DNS 8.8.8.8"

# 客户端互访
client-to-client

# 保持连接
keepalive 10 120

# 加密算法
cipher AES-256-GCM
auth SHA256

# 持久化
persist-key
persist-tun

# 日志
status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3
```

### 5.3 客户端配置文件

```conf
# client.ovpn
client
dev tun
proto udp

# 服务端公网 IP 和端口
remote <服务端公网IP> 1194

# 允许重连
resolv-retry infinite
nobind

# 持久化
persist-key
persist-tun

# 证书（内嵌）
<ca>
-----BEGIN CERTIFICATE-----
... CA 证书内容 ...
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
... 客户端证书内容 ...
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
... 客户端私钥内容 ...
-----END PRIVATE KEY-----
</key>

# 加密
cipher AES-256-GCM
auth SHA256

verb 3
```

### 5.4 启动与验证

```bash
# 启动服务端
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server

# 查看状态
sudo systemctl status openvpn@server
tail -f /var/log/openvpn/openvpn.log

# 客户端连接
sudo openvpn --config client.ovpn
```

---

## 六、云厂商 VPN 服务

### 6.1 AWS Site-to-Site VPN

```
本地数据中心 ←→ AWS VPC

配置流程：
  1. 创建 VPN 网关（Virtual Private Gateway）
  2. 创建客户网关（Customer Gateway，填写本地公网 IP）
  3. 创建 Site-to-Site VPN 连接
  4. 下载配置（支持多种本地设备）
  5. 配置本地路由器/防火墙
  6. 更新路由表

定价：按小时计费 + 数据传输费
```

### 6.2 阿里云 VPN 网关

```
配置流程：
  1. 创建 VPN 网关（绑定公网 IP）
  2. 创建用户网关（填写本地公网 IP）
  3. 创建 IPsec 连接（配置加密参数）
  4. 发布路由
  5. 本地设备配置 IKE 协商参数

支持：IPsec、SSL VPN
```

### 6.3 对比

| 云厂商 | 产品名称 | 协议 | 特点 |
|--------|---------|------|------|
| AWS | Site-to-Site VPN | IPsec | 与 Transit Gateway 深度集成 |
| 阿里云 | VPN 网关 | IPsec / SSL | 国内网络优化好 |
| 腾讯云 | VPN 连接 | IPsec | 支持 SPD 策略 |
| GCP | Cloud VPN | IPsec | 全球 VPC 互联 |

---

## 七、常见场景与最佳实践

### 场景 1：远程办公 — 访问公司内网

```
员工笔记本电脑（WireGuard 客户端）
    │
    │ 加密隧道（互联网）
    │
    ▼
公司 VPN 服务器（WireGuard）
    │
    ├── 内网 GitLab: 192.168.1.10
    ├── 内网 NAS: 192.168.1.20
    └── 内网 Jenkins: 192.168.1.30

配置要点：
  - 客户端 AllowedIPs = 192.168.1.0/24（只分流内网流量）
  - 服务端开启 NAT 转发
  - 每个员工一个密钥对，离职就删除 Peer
```

### 场景 2：多云互联 — 连接 AWS 和阿里云

```
AWS VPC (10.0.0.0/16)          阿里云 VPC (172.16.0.0/16)
    │                                │
    └── WireGuard 服务器 ──加密隧道── WireGuard 服务器 ──┘
         (AWS EC2)                    (阿里云 ECS)

配置要点：
  - 两端各部署一台 WireGuard 服务器
  - 两端都开启 IP 转发
  - 两端路由表互相指向对方
  - 不用云厂商 VPN 网关（贵），直接用 WireGuard（免费）
```

### 场景 3：访问海外资源

```
国内服务器
    │
    │ 加密隧道
    ▼
海外 VPN 服务器
    │
    ▼
访问 Google / GitHub / API

配置要点：
  - 客户端 AllowedIPs = 0.0.0.0/0（全局代理）
  - 或者用路由表分流（国内 IP 直连，国外走 VPN）
```

---

## 八、故障排查

### 8.1 WireGuard 排障

```bash
# 1. 检查接口状态
sudo wg show
# 看 latest handshake → 如果很久没握手，说明隧道断了

# 2. 检查密钥是否匹配
# 服务端 Peer 的 PublicKey = 客户端 Interface 的 PrivateKey 对应的公钥

# 3. 检查端口是否开放
sudo ss -tlnp | grep 51820    # 服务端是否监听
nc -vzu <服务端IP> 51820       # 客户端测试连通性

# 4. 检查防火墙
sudo iptables -L -n -v | grep 51820
sudo ufw status | grep 51820

# 5. 检查 IP 转发
sysctl net.ipv4.ip_forward    # 必须为 1

# 6. 抓包分析
sudo tcpdump -i wg0 -n
```

### 8.2 常见问题

| 问题 | 可能原因 | 解决 |
|------|---------|------|
| 握手成功但 ping 不通 | IP 转发未开启 | `sysctl -w net.ipv4.ip_forward=1` |
| 能 ping 通 VPN 服务器但 ping 不通内网 | NAT 规则缺失 | 添加 MASQUERADE 规则 |
| 手机切 WiFi 后断线 | 未设置 PersistentKeepalive | 设 `PersistentKeepalive = 25` |
| 端口不通 | 防火墙/安全组 | 开放 UDP 51820 |
| 客户端连不上 | 公私钥不匹配 | 重新生成密钥对 |

---

## 九、总结

| 维度 | 推荐方案 |
|------|---------|
| **个人/小团队** | WireGuard — 配置 5 分钟，性能最强 |
| **企业级** | OpenVPN — 功能全面，证书管理成熟 |
| **移动端** | IKEv2 — 网络切换不掉线 |
| **云厂商互联** | WireGuard 自建（不用云 VPN 网关，省钱） |
| **跨国访问** | WireGuard 或 Shadowsocks + 插件 |

> **一句话总结**：VPN 就是在不安全的互联网上开一条加密隧道，让远程设备像在局域网一样访问私有资源。WireGuard 是目前最好的选择——配置简单、性能高、安全性强。