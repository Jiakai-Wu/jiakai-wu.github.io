---
title: Tailscale组网及私有化使用
date: 2025-06-25 19:14:15 +0800
categories: [VPN, WireGuard]
tags: [VPN, WireGuard]
---
### 什么是Tailscale？

> 前言:
> 家里有4070 Ti super显卡在家吃灰，希望能利用这个资源来跑大模型，需要远程访问，但需要在公司访问时候方便快捷。
> 考虑到需要远程访问需要内网穿透的方式实现，但内网穿透的带宽太小，ComfyUI也需要特别要求久，最后选择tailscale组建derp中继服务达到较好的效果。

Tailscale 是一个基于 WireGuard 协议构建的现代化网络安全工具，旨在通过零配置实现设备间的安全点对点连接。

### 核心功能与技术架构

#### 零配置网络架构

Tailscale 不同于传统 VPN 需要配置 IP 地址，通过集成 Google、Microsoft 等服务提供商，实现基于用户/组的动态权限管理，支持细粒度访问控制（例如限制特定用户访问数据库端口）。

#### WireGuard 协议优化

基于 WireGuard 的高效加密机制，比传统 VPN（如 IPSec）资源占用更低，延迟更小，内核态实现进一步提升性能，适合高并发场景。

#### 动态网络拓扑

自动构建网状网络（Mesh），无需中心节点。设备间通过 DERP 服务器中继或直接 P2P 连接，适应复杂网络环境（如 NAT 穿透）。

#### 跨路由器及跨平台支持

支持跨路由器组网（如 OpenWrt），实现整个网络接入 Tailscale 网络。支持 macOS、Windows、Linux、iOS/Android 等操作系统，并提供官方客户端和第三方插件（如 OpenWrt 的 Luci 插件）

### 典型使用场景

#### 远程办公与内网穿透

安全访问家庭 NAS、企业内网服务，无需暴露公网 IP。例如，通过 Tailscale IP 直接 SSH 连接到办公室或家庭设备。

#### 团队协作与运维协同

开发者可快速建立临时测试环境，运维人员通过统一网络管理后台，批量支持 SSH 密钥远程分发和会话记录。

#### 替代传统 VPN

简化配置流程，避免传统 VPN 的复杂密钥管理和端口开放问题。适合替代小团队的 OpenVPN 或 IPSec。

#### 多云环境部署

连接不同云厂商（如 AWS、阿里云）的服务器，构建私有网络，避免公网暴露风险。

### 安装与配置

#### 1. 安装 Tailscale 客户端

**Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

**macOS:**
```bash
brew install tailscale
```

**Windows:**
从官网下载安装包：https://tailscale.com/download

**OpenWrt:**
```bash
opkg update
opkg install tailscale
```

#### 2. 启动并登录

```bash
# 启动 Tailscale
sudo tailscale up

# 查看状态
tailscale status

# 查看本机 Tailscale IP
tailscale ip -4
```

首次运行会提示登录，使用 Google、Microsoft 或 GitHub 账号授权。

#### 3. 配置 ACL 访问控制

在 Tailscale 管理后台（https://login.tailscale.com/admin/acls）配置访问策略：

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:developers"],
      "dst": ["tag:production:22,80,443"]
    },
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["*:*"]
    }
  ]
}
```

#### 4. 配置子网路由（可选）

如果需要让整个局域网接入 Tailscale：

```bash
# 在路由器或网关设备上启用子网路由
sudo tailscale up --advertise-routes=192.168.1.0/24

# 在管理后台批准子网路由
```

### 私有化部署 DERP 中继服务器

#### 为什么需要自建 DERP？

- **降低延迟**：官方 DERP 服务器可能距离较远
- **提高带宽**：自建服务器可以使用更高带宽
- **数据隐私**：避免流量经过第三方服务器
- **稳定性**：避免官方服务器故障影响

#### 部署步骤

**1. 准备服务器**
- 一台有公网 IP 的 Linux 服务器（推荐 Ubuntu 20.04+）
- 开放端口：443（HTTPS）、3478（STUN）

**2. 安装 Tailscale**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**3. 部署 DERP 服务**

创建配置文件 `/etc/derp.conf`:
```json
{
  "RegionID": 900,
  "RegionCode": "myderp",
  "RegionName": "My DERP Server",
  "Nodes": [
    {
      "Name": "myderp-1",
      "RegionID": 900,
      "HostName": "derp.example.com",
      "DERPPort": 443,
      "STUNPort": 3478
    }
  ]
}
```

**4. 启动 DERP 服务**
```bash
# 下载 derper 二进制文件
wget https://pkgs.tailscale.com/stable/derper
chmod +x derper

# 使用 systemd 管理服务
sudo tee /etc/systemd/system/derper.service > /dev/null <<EOF
[Unit]
Description=Tailscale DERP Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/derper -hostname=derp.example.com -a :443 -stun -verify-clients
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable derper
sudo systemctl start derper
```

**5. 配置 SSL 证书（使用 Let's Encrypt）**
```bash
sudo apt install certbot
sudo certbot certonly --standalone -d derp.example.com
```

**6. 在 Tailscale 管理后台配置自定义 DERP**

在 ACL 配置中添加：
```json
{
  "derpMap": {
    "Regions": {
      "900": {
        "RegionID": 900,
        "RegionCode": "myderp",
        "RegionName": "My DERP Server",
        "Nodes": [
          {
            "Name": "myderp-1",
            "RegionID": 900,
            "HostName": "derp.example.com",
            "DERPPort": 443,
            "STUNPort": 3478
          }
        ]
      }
    }
  }
}
```

#### 验证 DERP 服务

```bash
# 测试 DERP 连接
curl https://derp.example.com/derp/latency-check

# 查看 Tailscale 使用的 DERP 服务器
tailscale netcheck
```

### 高级配置

#### 1. 配置 Exit Node（出口节点）

让所有流量通过指定设备路由：

```bash
# 在出口节点设备上
sudo tailscale up --advertise-exit-node

# 在客户端设备上
sudo tailscale up --exit-node=<exit-node-ip>
```

#### 2. 配置 MagicDNS

自动为 Tailscale 设备分配域名：

```bash
# 在管理后台启用 MagicDNS
# 设备可以通过 hostname.tailnet-name.ts.net 访问
```

#### 3. 配置 Taildrop（文件传输）

```bash
# 发送文件
tailscale file cp myfile.txt <target-device>:

# 接收文件
tailscale file get
```

### 性能优化建议

1. **启用 BBR 拥塞控制**（Linux）：
```bash
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

2. **调整 MTU 值**：
```bash
sudo tailscale up --mtu=1280
```

3. **禁用不必要的 DERP 区域**：
在 ACL 中只保留需要的 DERP 服务器。

### 常见问题排查

#### 无法建立 P2P 连接

```bash
# 检查 NAT 类型
tailscale netcheck

# 强制使用 DERP 中继
sudo tailscale up --force-reauth
```

#### DERP 服务器连接失败

```bash
# 检查防火墙规则
sudo ufw allow 443/tcp
sudo ufw allow 3478/udp

# 查看 DERP 服务日志
sudo journalctl -u derper -f
```

#### 延迟过高

```bash
# 测试到各个 DERP 服务器的延迟
tailscale netcheck

# 切换到延迟更低的 DERP 区域
```

### 安全最佳实践

1. **启用 MFA（多因素认证）**：在 Tailscale 管理后台强制要求 MFA
2. **定期审查 ACL 规则**：确保访问权限最小化
3. **使用 Key Expiry**：设置设备密钥过期时间
4. **启用审计日志**：记录所有网络访问行为
5. **隔离敏感设备**：使用 Tag 和 ACL 隔离生产环境

### 总结

Tailscale 通过 WireGuard 协议和零配置设计，极大简化了组网复杂度。自建 DERP 服务器可以进一步提升性能和隐私保护。无论是个人远程访问、团队协作还是多云环境部署，Tailscale 都是一个值得尝试的现代化网络解决方案。

### 参考资料

1. [Tailscale 官方文档](https://tailscale.com/kb/)
2. [WireGuard 协议规范](https://www.wireguard.com/)
3. [自建 DERP 服务器指南](https://tailscale.com/kb/1118/custom-derp-servers/)
