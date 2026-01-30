---
title: Tailscale DERP 服务器部署 Moltbot 远程操作本地主机指南
date: 2026-01-30 22:41:00 +0800
categories: [AI, Network]
tags: [Moltbot, Tailscale, DERP, Remote, Automation]
---

> 前言：通过 Tailscale 组建虚拟局域网，可以让部署在云服务器上的 Moltbot 安全地访问和操作本地主机。本文将详细介绍如何在 Tailscale DERP 服务器上部署 Moltbot，并实现对本地主机的远程控制。

### 应用场景

#### 为什么需要这种架构？

**典型场景：**
- QQ 机器人需要固定公网 IP（白名单限制）
- 本地主机有强大的计算资源（如 GPU）
- 希望 Moltbot 能同时访问云端和本地资源
- 需要安全的内网穿透方案

**架构优势：**
- ✅ 云服务器提供稳定的公网接入
- ✅ Tailscale 提供安全的内网连接
- ✅ 本地主机保留强大的计算能力
- ✅ 无需暴露本地主机到公网

### 架构设计

```
QQ 平台
   ↓
云服务器（公网IP）
   ├─ Tailscale DERP 服务
   ├─ Moltbot Gateway
   └─ Tailscale 客户端
        ↓ (虚拟内网)
本地主机（动态IP）
   ├─ Tailscale 客户端
   ├─ 计算资源（GPU/大内存）
   └─ 本地服务（API/数据库等）
```

**数据流向：**
1. QQ 消息 → 云服务器（公网）
2. Moltbot 处理 → 判断是否需要本地资源
3. 通过 Tailscale → 访问本地主机（内网）
4. 本地主机执行 → 返回结果
5. Moltbot 整合 → 回复 QQ 消息

### 环境准备

#### 云服务器要求

**硬件配置：**
- CPU：2核及以上
- 内存：2GB 及以上（推荐 4GB）
- 磁盘：20GB 及以上
- 带宽：5Mbps 及以上

**软件要求：**
- 操作系统：Ubuntu 20.04/22.04 LTS
- 公网 IP：固定 IP（用于 QQ 白名单）
- 开放端口：443（HTTPS）、3478（STUN）

#### 本地主机要求

**硬件配置：**
- 根据实际需求配置（如需要 GPU 推理）
- 稳定的网络连接

**软件要求：**
- 操作系统：Windows 10/11 或 Linux
- 可以访问互联网

### 第一步：部署 Tailscale DERP 服务器

#### 1. 安装 Tailscale

```bash
# 在云服务器上执行
curl -fsSL https://tailscale.com/install.sh | sh

# 启动 Tailscale
sudo tailscale up

# 查看 Tailscale IP
tailscale ip -4
# 输出示例：100.64.0.1
```

#### 2. 安装 DERP 服务

```bash
# 下载 derper 二进制文件
sudo wget -O /usr/local/bin/derper https://pkgs.tailscale.com/stable/derper
sudo chmod +x /usr/local/bin/derper

# 创建配置目录
sudo mkdir -p /etc/derp
```

#### 3. 配置 SSL 证书

```bash
# 安装 certbot
sudo apt update
sudo apt install -y certbot

# 申请证书（替换为你的域名）
sudo certbot certonly --standalone -d derp.example.com

# 证书路径
# /etc/letsencrypt/live/derp.example.com/fullchain.pem
# /etc/letsencrypt/live/derp.example.com/privkey.pem
```

#### 4. 创建 DERP 服务

创建 systemd 服务文件 `/etc/systemd/system/derper.service`：

```ini
[Unit]
Description=Tailscale DERP Server
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/derper \
  -hostname=derp.example.com \
  -a :443 \
  -stun \
  -certmode=manual \
  -certdir=/etc/letsencrypt/live/derp.example.com \
  -verify-clients
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable derper
sudo systemctl start derper
sudo systemctl status derper
```

#### 5. 配置防火墙

```bash
# 开放端口
sudo ufw allow 443/tcp
sudo ufw allow 3478/udp
sudo ufw enable
```

#### 6. 验证 DERP 服务

```bash
# 测试 DERP 连接
curl https://derp.example.com/derp/latency-check

# 应该返回类似：
# {"RegionID":900,"RegionCode":"custom","Latency":"1.2ms"}
```

#### 7. 配置自定义 DERP

在 Tailscale 管理后台（login.tailscale.com/admin/acls）添加：

```json
{
  "derpMap": {
    "OmitDefaultRegions": false,
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

### 第二步：在云服务器上部署 Moltbot

#### 1. 安装 Node.js

```bash
# 安装 Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 验证安装
node --version
npm --version
```

#### 2. 安装 Moltbot

```bash
# 全局安装 Moltbot
sudo npm install -g moltbot

# 验证安装
moltbot --version
```

#### 3. 创建工作目录

```bash
# 创建 Moltbot 工作目录
mkdir -p ~/moltbot-workspace
cd ~/moltbot-workspace

# 初始化配置
moltbot init
```

#### 4. 配置 Moltbot

编辑 `.moltbot/config.yaml`：

```yaml
# Moltbot 主配置
agents:
  defaults:
    model: anthropic/claude-sonnet-4-5
    thinking: off
    temperature: 0.7

# 配置 QQBot
channels:
  qqbot:
    enabled: true
    appId: "你的AppID"
    token: "你的Token"
    secret: "你的Secret"
    sandbox: false
    intents:
      - public_messages
      - direct_messages

# 配置模型 API
providers:
  anthropic:
    apiKey: "sk-ant-xxxxx"
    # 如果需要代理
    # baseURL: "https://your-proxy.com"

# 安全配置
security:
  exec:
    mode: allowlist
    allowlist:
      - "ls"
      - "cat"
      - "echo"
      - "curl"
      - "ssh"
```

#### 5. 配置环境变量

```bash
# 编辑 ~/.bashrc 或 ~/.zshrc
export QQBOT_APP_ID="your_app_id"
export QQBOT_TOKEN="your_token"
export QQBOT_SECRET="your_secret"
export ANTHROPIC_API_KEY="sk-ant-xxxxx"

# 重新加载配置
source ~/.bashrc
```

#### 6. 启动 Moltbot

```bash
# 启动 Gateway
moltbot gateway start

# 查看状态
moltbot gateway status

# 查看日志
moltbot gateway logs -f
```

### 第三步：配置本地主机

#### Windows 本地主机

**1. 安装 Tailscale**

下载并安装 Tailscale Windows 客户端：
- 访问 tailscale.com/download
- 下载 Windows 安装包
- 运行安装程序
- 登录 Tailscale 账号

**2. 验证连接**

```powershell
# 查看 Tailscale 状态
tailscale status

# 查看本地 Tailscale IP
tailscale ip -4
# 输出示例：100.64.0.2

# 测试连接云服务器
ping 100.64.0.1
```

**3. 配置 SSH 服务（可选）**

如果需要 Moltbot 通过 SSH 操作本地主机：

```powershell
# 启用 OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 启动服务
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'

# 配置防火墙
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

#### Linux 本地主机

**1. 安装 Tailscale**

```bash
# 安装 Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# 启动并登录
sudo tailscale up

# 查看 Tailscale IP
tailscale ip -4
```

**2. 配置 SSH**

```bash
# 确保 SSH 服务运行
sudo systemctl enable ssh
sudo systemctl start ssh

# 配置 SSH 密钥认证（推荐）
# 在云服务器上生成密钥对
ssh-keygen -t ed25519 -C "moltbot@cloud"

# 将公钥复制到本地主机
ssh-copy-id user@100.64.0.2
```

### 第四步：配置 Moltbot 访问本地主机

#### 1. 配置 SSH 连接

在云服务器的 Moltbot 工作目录创建 `TOOLS.md`：

```markdown
# TOOLS.md - Local Notes

## 本地主机配置

### SSH 连接信息
- **主机名**：local-workstation
- **Tailscale IP**：100.64.0.2
- **SSH 端口**：22
- **用户名**：your-username
- **认证方式**：密钥认证

### 常用命令
```bash
# 连接本地主机
ssh user@100.64.0.2

# 执行远程命令
ssh user@100.64.0.2 "command"
```

### 本地服务
- **GPU 推理服务**：http://100.64.0.2:8000
- **本地数据库**：100.64.0.2:5432
- **文件共享**：smb://100.64.0.2/share
```

#### 2. 测试连接

在云服务器上测试：

```bash
# 测试 Tailscale 连接
ping 100.64.0.2

# 测试 SSH 连接
ssh user@100.64.0.2 "echo 'Connection successful'"

# 测试本地服务
curl http://100.64.0.2:8000/health
```

### 第五步：使用 Moltbot 操作本地主机

#### 基本操作示例

**1. 查询本地主机信息**

在 QQ 中发送：
```
查看本地主机的系统信息
```

Moltbot 会执行：
```bash
ssh user@100.64.0.2 "uname -a && df -h && free -h"
```

**2. 执行本地命令**

```
在本地主机上运行 nvidia-smi
```

Moltbot 会执行：
```bash
ssh user@100.64.0.2 "nvidia-smi"
```

**3. 文件操作**

```
从本地主机下载 /home/user/report.txt
```

Moltbot 会执行：
```bash
scp user@100.64.0.2:/home/user/report.txt /tmp/
cat /tmp/report.txt
```

**4. 调用本地 API**

```
调用本地 GPU 服务生成图片
```

Moltbot 会执行：
```bash
curl -X POST http://100.64.0.2:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "a beautiful sunset"}'
```

#### 高级操作示例

**1. 自动化脚本执行**

在 QQ 中发送：
```
在本地主机上运行数据备份脚本
```

Moltbot 会：
1. SSH 连接到本地主机
2. 执行备份脚本
3. 监控执行进度
4. 返回执行结果

**2. 多步骤任务**

```
帮我在本地主机上：
1. 拉取最新代码
2. 运行测试
3. 如果测试通过，部署到生产环境
```

Moltbot 会自动规划并执行：
```bash
ssh user@100.64.0.2 "cd /project && git pull"
ssh user@100.64.0.2 "cd /project && npm test"
# 如果测试通过
ssh user@100.64.0.2 "cd /project && npm run deploy"
```

**3. 监控和告警**

配置定时任务监控本地主机：

```yaml
# 在 .moltbot/config.yaml 中添加
cron:
  jobs:
    - name: "监控本地主机"
      schedule:
        kind: cron
        expr: "*/5 * * * *"  # 每5分钟
        tz: "Asia/Shanghai"
      payload:
        kind: systemEvent
        text: "检查本地主机状态"
      sessionTarget: main
```

Moltbot 会定期检查：
- CPU/内存使用率
- 磁盘空间
- GPU 状态
- 服务运行状态

如果发现异常，自动发送 QQ 消息告警。

### 安全配置

#### 1. SSH 密钥管理

**生成专用密钥：**

```bash
# 在云服务器上
ssh-keygen -t ed25519 -f ~/.ssh/moltbot_local -C "moltbot-access"

# 配置 SSH config
cat >> ~/.ssh/config << EOF
Host local-workstation
    HostName 100.64.0.2
    User your-username
    IdentityFile ~/.ssh/moltbot_local
    StrictHostKeyChecking yes
EOF

# 将公钥添加到本地主机
ssh-copy-id -i ~/.ssh/moltbot_local.pub user@100.64.0.2
```

#### 2. 限制 SSH 权限

在本地主机上配置 `/etc/ssh/sshd_config`：

```bash
# 只允许特定用户
AllowUsers moltbot-user

# 禁用密码认证
PasswordAuthentication no

# 只允许密钥认证
PubkeyAuthentication yes

# 限制登录尝试
MaxAuthTries 3

# 重启 SSH 服务
sudo systemctl restart sshd
```

#### 3. 配置防火墙规则

**本地主机（Linux）：**

```bash
# 只允许 Tailscale 网络访问 SSH
sudo ufw allow from 100.64.0.0/10 to any port 22
sudo ufw deny 22
sudo ufw enable
```

**本地主机（Windows）：**

```powershell
# 只允许 Tailscale IP 访问 SSH
New-NetFirewallRule -DisplayName "SSH from Tailscale" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 22 `
  -RemoteAddress 100.64.0.0/10 `
  -Action Allow
```

#### 4. 审计日志

在云服务器上配置 Moltbot 审计：

```yaml
# .moltbot/config.yaml
logging:
  level: info
  audit: true
  file: /var/log/moltbot/audit.log
  
  # 记录所有 SSH 命令
  auditCommands:
    - ssh
    - scp
    - rsync
```

### 性能优化

#### 1. 启用 BBR 拥塞控制

在云服务器和本地主机上：

```bash
# 启用 BBR
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 验证
sysctl net.ipv4.tcp_congestion_control
```

#### 2. 优化 SSH 连接

在云服务器的 `~/.ssh/config` 中：

```bash
Host local-workstation
    HostName 100.64.0.2
    User your-username
    IdentityFile ~/.ssh/moltbot_local
    
    # 连接复用
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 10m
    
    # 保持连接
    ServerAliveInterval 60
    ServerAliveCountMax 3
    
    # 压缩传输
    Compression yes
```

创建 socket 目录：

```bash
mkdir -p ~/.ssh/sockets
```

#### 3. 使用连接池

对于频繁的 SSH 操作，保持长连接：

```bash
# 建立持久连接
ssh -M -S ~/.ssh/sockets/local-workstation -fNT local-workstation

# 后续命令复用连接
ssh -S ~/.ssh/sockets/local-workstation local-workstation "command"
```

### 监控与维护

#### 1. 监控 Tailscale 连接

```bash
# 查看 Tailscale 状态
tailscale status

# 查看网络延迟
tailscale netcheck

# 查看连接详情
tailscale ping 100.64.0.2
```

#### 2. 监控 DERP 服务

```bash
# 查看 DERP 日志
sudo journalctl -u derper -f

# 查看连接统计
curl https://derp.example.com/derp/stats
```

#### 3. 监控 Moltbot

```bash
# 查看 Moltbot 状态
moltbot gateway status

# 查看实时日志
moltbot gateway logs -f

# 查看会话列表
moltbot sessions list
```

#### 4. 自动化监控脚本

创建 `/usr/local/bin/monitor-moltbot.sh`：

```bash
#!/bin/bash

# 检查 Tailscale 连接
if ! tailscale status | grep -q "100.64.0.2"; then
    echo "Tailscale connection to local host failed"
    # 发送告警
fi

# 检查 DERP 服务
if ! systemctl is-active --quiet derper; then
    echo "DERP service is down"
    sudo systemctl restart derper
fi

# 检查 Moltbot
if ! moltbot gateway status | grep -q "running"; then
    echo "Moltbot is down"
    moltbot gateway restart
fi

# 检查本地主机连接
if ! ssh -o ConnectTimeout=5 local-workstation "echo ok" &>/dev/null; then
    echo "Cannot connect to local host"
fi
```

添加到 crontab：

```bash
# 每5分钟检查一次
*/5 * * * * /usr/local/bin/monitor-moltbot.sh
```

### 故障排查

#### 1. Tailscale 连接失败

**检查 Tailscale 状态：**

```bash
# 云服务器
tailscale status
tailscale netcheck

# 本地主机
tailscale status
tailscale netcheck
```

**常见问题：**
- 防火墙阻止：检查 443 和 3478 端口
- DERP 服务未运行：`sudo systemctl status derper`
- 证书过期：重新申请 Let's Encrypt 证书

#### 2. SSH 连接失败

**测试连接：**

```bash
# 详细模式
ssh -vvv user@100.64.0.2

# 检查密钥权限
ls -la ~/.ssh/moltbot_local
# 应该是 600
```

**常见问题：**
- 密钥权限错误：`chmod 600 ~/.ssh/moltbot_local`
- 本地主机 SSH 服务未运行
- 防火墙阻止 22 端口

#### 3. Moltbot 无法执行命令

**检查配置：**

```yaml
# .moltbot/config.yaml
security:
  exec:
    mode: allowlist
    allowlist:
      - "ssh"  # 确保 ssh 在白名单中
```

**查看日志：**

```bash
moltbot gateway logs | grep "exec"
```

### 生产环境部署

#### 1. 使用 systemd 管理 Moltbot

创建 `/etc/systemd/system/moltbot.service`：

```ini
[Unit]
Description=Moltbot Gateway
After=network.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=simple
User=moltbot
WorkingDirectory=/home/moltbot/moltbot-workspace
Environment="NODE_ENV=production"
ExecStart=/usr/bin/moltbot gateway start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable moltbot
sudo systemctl start moltbot
```

#### 2. 配置日志轮转

创建 `/etc/logrotate.d/moltbot`：

```
/var/log/moltbot/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 moltbot moltbot
    sharedscripts
    postrotate
        systemctl reload moltbot > /dev/null 2>&1 || true
    endscript
}
```

#### 3. 配置自动备份

创建备份脚本 `/usr/local/bin/backup-moltbot.sh`：

```bash
#!/bin/bash

BACKUP_DIR="/backup/moltbot"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

# 备份配置和数据
tar -czf $BACKUP_DIR/moltbot-$DATE.tar.gz \
    /home/moltbot/moltbot-workspace/.moltbot \
    /home/moltbot/moltbot-workspace/*.md \
    /home/moltbot/moltbot-workspace/memory

# 保留最近7天的备份
find $BACKUP_DIR -name "moltbot-*.tar.gz" -mtime +7 -delete
```

添加到 crontab：

```bash
# 每天凌晨2点备份
0 2 * * * /usr/local/bin/backup-moltbot.sh
```

### 总结

通过本文，你已经学会了：

1. ✅ 部署 Tailscale DERP 服务器
2. ✅ 在云服务器上部署 Moltbot
3. ✅ 配置本地主机接入 Tailscale
4. ✅ 实现 Moltbot 远程操作本地主机
5. ✅ 安全配置和权限管理
6. ✅ 性能优化和监控维护
7. ✅ 生产环境部署

这种架构结合了云服务器的稳定性和本地主机的计算能力，通过 Tailscale 提供安全的内网连接，是一个理想的 AI 助手部署方案。

### 参考资料

1. Tailscale 官方文档：docs.tailscale.com
2. Moltbot 官方文档：docs.molt.bot
3. 自建 DERP 服务器指南：tailscale.com/kb/1118/custom-derp-servers
4. SSH 安全配置最佳实践：www.ssh.com/academy/ssh/config
5. systemd 服务管理：www.freedesktop.org/software/systemd/man/systemd.service.html
