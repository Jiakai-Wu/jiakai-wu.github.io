---
title: Moltbot 本地安装与 QQBot 插件配置指南
date: 2026-01-30 21:47:00 +0800
categories: [AI, Chatbot]
tags: [Moltbot, QQBot, AI, Automation]
---

> 前言：Moltbot 是一个强大的 AI 助手框架，支持多平台消息接入（QQ、Telegram、Discord 等）。本文将详细介绍如何在本地安装 Moltbot，配置 QQBot 插件，并实现 QQ 机器人的自动化操作。

### 什么是 Moltbot？

Moltbot 是一个基于大语言模型（LLM）的智能助手框架，具有以下特点：

- **多平台支持**：QQ、Telegram、WhatsApp、Discord、Signal 等
- **工具调用能力**：支持 Function Call、文件操作、浏览器控制等
- **会话管理**：支持多会话隔离、子任务派发
- **可扩展性**：插件化架构，易于扩展新功能
- **本地部署**：支持完全本地化部署，保护隐私

### 系统要求

#### 硬件要求
- **CPU**：2核及以上
- **内存**：4GB 及以上（推荐 8GB）
- **磁盘**：至少 2GB 可用空间

#### 软件要求
- **操作系统**：Windows 10/11、Linux（Ubuntu 20.04+）、macOS 10.15+
- **Node.js**：v18.0.0 及以上版本
- **Git**：用于版本管理（可选）

### 安装 Node.js

#### Windows

1. 访问 Node.js 官网下载安装包
2. 运行安装程序，按默认选项安装
3. 验证安装：
```cmd
node --version
npm --version
```

#### Linux (Ubuntu/Debian)

```bash
# 使用 NodeSource 仓库安装最新版本
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 验证安装
node --version
npm --version
```

#### macOS

```bash
# 使用 Homebrew 安装
brew install node

# 验证安装
node --version
npm --version
```

### 安装 Moltbot

#### 方法一：全局安装（推荐）

```bash
# 使用 npm 全局安装
npm install -g moltbot

# 验证安装
moltbot --version
```

#### 方法二：使用 npx（无需安装）

```bash
# 直接运行，无需全局安装
npx moltbot --version
```

### 初始化 Moltbot

#### 1. 创建工作目录

```bash
# 创建并进入工作目录
mkdir moltbot-workspace
cd moltbot-workspace

# 初始化配置
moltbot init
```

初始化过程会创建以下文件结构：

```
moltbot-workspace/
├── .moltbot/
│   ├── config.yaml          # 主配置文件
│   └── data/                # 数据目录
├── AGENTS.md                # 助手行为指南
├── SOUL.md                  # 助手人格定义
├── USER.md                  # 用户信息
├── MEMORY.md                # 长期记忆
├── TOOLS.md                 # 工具配置
└── memory/                  # 每日记忆文件
```

#### 2. 配置基本信息

编辑 `USER.md`，填写你的基本信息：

```markdown
# USER.md - About Your Human

- **Name:** 你的名字
- **What to call them:** 称呼
- **Pronouns:** 代词（可选）
- **Timezone:** Asia/Shanghai
- **Notes:** 其他备注
```

编辑 `IDENTITY.md`，定义助手身份：

```markdown
# IDENTITY.md - Who Am I?

- **Name:** 小助手
- **Creature:** AI助手
- **Vibe:** 友好、专业、高效
- **Emoji:** 🤖
```

### 配置 QQBot 插件

#### 1. 获取 QQ 机器人凭证

访问 QQ 开放平台（q.qq.com），创建机器人应用：

1. 登录 QQ 开放平台
2. 创建应用 → 选择"机器人"类型
3. 获取以下信息：
   - **AppID**：应用ID
   - **Token**：机器人令牌
   - **Secret**：应用密钥

#### 2. 配置白名单 IP

在 QQ 开放平台的机器人配置页面：

1. 找到"服务器配置"
2. 添加你的服务器公网 IP 到白名单
3. 如果是本地测试，可以使用内网穿透工具（如 ngrok、frp）

#### 3. 编辑 Moltbot 配置文件

编辑 `.moltbot/config.yaml`，添加 QQBot 配置：

```yaml
# Moltbot 主配置文件
agents:
  defaults:
    model: anthropic/claude-sonnet-4-5  # 默认模型
    thinking: off                        # 推理模式
    temperature: 0.7

channels:
  qqbot:
    enabled: true
    appId: "你的AppID"
    token: "你的Token"
    secret: "你的Secret"
    sandbox: false                       # 是否为沙箱环境
    intents:
      - public_messages                  # 接收公域消息
      - direct_messages                  # 接收私信
      - guild_messages                   # 接收频道消息
    # 可选配置
    maxRetries: 3                        # 最大重试次数
    timeout: 30000                       # 超时时间（毫秒）
```

#### 4. 配置模型 API

Moltbot 支持多种 LLM 提供商，需要配置至少一个：

**Anthropic (Claude)：**

```yaml
providers:
  anthropic:
    apiKey: "sk-ant-xxxxx"
    baseURL: "https://api.anthropic.com"  # 可选，使用代理时修改
```

**OpenAI (GPT)：**

```yaml
providers:
  openai:
    apiKey: "sk-xxxxx"
    baseURL: "https://api.openai.com/v1"
```

**本地模型（Ollama）：**

```yaml
providers:
  ollama:
    baseURL: "http://localhost:11434"
    models:
      - llama3
      - qwen2
```

### 启动 Moltbot

#### 1. 启动 Gateway 服务

```bash
# 启动 Moltbot Gateway（后台服务）
moltbot gateway start

# 查看运行状态
moltbot gateway status

# 查看日志
moltbot gateway logs

# 停止服务
moltbot gateway stop

# 重启服务
moltbot gateway restart
```

#### 2. 验证 QQBot 连接

启动成功后，日志中应该看到类似输出：

```
[INFO] QQBot channel initialized
[INFO] Connected to QQ Open Platform
[INFO] Bot ready: 机器人名称 (AppID: xxxxx)
```

#### 3. 测试机器人

在 QQ 中：
1. 搜索你的机器人名称
2. 发送消息："你好"
3. 机器人应该会回复

### QQBot 使用指南

#### 基本对话

直接在 QQ 中与机器人对话：

```
用户：你好
机器人：你好！我是你的AI助手，有什么可以帮你的吗？

用户：今天天气怎么样？
机器人：抱歉，我目前无法直接查询天气。你可以告诉我你的位置，我可以帮你搜索天气信息。
```

#### 文件操作

机器人可以读取、创建、编辑文件：

```
用户：帮我创建一个 todo.txt 文件，内容是"1. 学习 Moltbot"
机器人：已创建文件 todo.txt

用户：读取 todo.txt
机器人：文件内容：
1. 学习 Moltbot
```

#### 代码执行

机器人可以执行 shell 命令（需谨慎使用）：

```
用户：帮我查看当前目录下的文件
机器人：[执行 ls 命令]
当前目录文件：
- config.yaml
- AGENTS.md
- SOUL.md
```

#### 网络搜索

如果配置了搜索 API（如 Brave Search），机器人可以搜索网络：

```
用户：搜索一下 Moltbot 的最新版本
机器人：[搜索中...]
根据搜索结果，Moltbot 最新版本是 v1.2.3...
```

#### 多轮对话

机器人会记住上下文：

```
用户：我叫张三
机器人：你好，张三！很高兴认识你。

用户：我叫什么名字？
机器人：你叫张三。
```

### 高级配置

#### 1. 配置访问控制（ACL）

限制哪些用户可以使用机器人：

```yaml
channels:
  qqbot:
    acl:
      allowlist:
        - "123456789"  # 允许的 QQ 号
        - "987654321"
      denylist:
        - "111111111"  # 禁止的 QQ 号
```

#### 2. 配置消息过滤

过滤敏感词或垃圾消息：

```yaml
channels:
  qqbot:
    filters:
      keywords:
        - "广告"
        - "spam"
      minLength: 2      # 最小消息长度
      maxLength: 2000   # 最大消息长度
```

#### 3. 配置自动回复

设置特定关键词的自动回复：

```yaml
channels:
  qqbot:
    autoReply:
      - trigger: "帮助"
        response: "我是AI助手，可以帮你：\n1. 回答问题\n2. 执行任务\n3. 搜索信息"
      - trigger: "联系管理员"
        response: "管理员QQ：123456789"
```

#### 4. 配置定时任务

使用 cron 功能设置定时提醒：

```bash
# 通过 QQ 发送命令
用户：每天早上9点提醒我开会
机器人：已设置定时提醒
```

或者直接编辑配置文件：

```yaml
cron:
  jobs:
    - name: "每日提醒"
      schedule:
        kind: cron
        expr: "0 9 * * *"
        tz: "Asia/Shanghai"
      payload:
        kind: systemEvent
        text: "早上好！今天有重要会议，记得准备材料。"
      sessionTarget: main
```

#### 5. 配置子任务派发

让机器人在后台执行复杂任务：

```
用户：帮我分析一下这个项目的代码质量（后台执行）
机器人：好的，我会在后台分析，完成后通知你。

[10分钟后]
机器人：代码质量分析完成！发现以下问题：...
```

### 常见问题排查

#### 1. 机器人无法启动

**检查配置文件：**
```bash
# 验证配置文件语法
moltbot gateway config.get
```

**检查日志：**
```bash
moltbot gateway logs
```

**常见错误：**
- `Invalid AppID/Token`：检查 QQ 开放平台凭证是否正确
- `Connection refused`：检查网络连接和防火墙设置
- `IP not in whitelist`：检查 QQ 开放平台的 IP 白名单配置

#### 2. 机器人不回复消息

**检查 intents 配置：**
```yaml
intents:
  - public_messages   # 必须包含
  - direct_messages   # 如果需要接收私信
```

**检查权限：**
- 确保机器人有发送消息的权限
- 检查是否被 QQ 限流

**查看日志：**
```bash
moltbot gateway logs | grep "qqbot"
```

#### 3. 消息发送失败

**检查 QQ 平台限制：**
- QQ 机器人有消息频率限制（通常为 5条/秒）
- 消息长度限制（通常为 4096 字符）
- 不支持某些特殊字符或格式

**解决方案：**
- 使用消息分片发送长消息
- 添加延迟避免频率限制
- 过滤特殊字符

#### 4. 模型 API 调用失败

**检查 API Key：**
```yaml
providers:
  anthropic:
    apiKey: "sk-ant-xxxxx"  # 确保正确
```

**检查网络：**
```bash
# 测试 API 连接
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01"
```

**使用代理：**
```yaml
providers:
  anthropic:
    baseURL: "https://your-proxy.com"
```

### 性能优化

#### 1. 使用本地模型

使用 Ollama 运行本地模型，降低 API 成本：

```bash
# 安装 Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型
ollama pull llama3

# 配置 Moltbot
```

```yaml
agents:
  defaults:
    model: ollama/llama3
```

#### 2. 配置缓存

启用响应缓存，减少重复请求：

```yaml
cache:
  enabled: true
  ttl: 3600  # 缓存时间（秒）
```

#### 3. 限制并发

避免同时处理过多请求：

```yaml
channels:
  qqbot:
    concurrency: 5  # 最大并发数
```

### 安全最佳实践

#### 1. 保护敏感信息

**不要在配置文件中硬编码密钥：**

```bash
# 使用环境变量
export QQBOT_APP_ID="your_app_id"
export QQBOT_TOKEN="your_token"
export QQBOT_SECRET="your_secret"
```

```yaml
channels:
  qqbot:
    appId: ${QQBOT_APP_ID}
    token: ${QQBOT_TOKEN}
    secret: ${QQBOT_SECRET}
```

#### 2. 限制命令执行

禁止执行危险命令：

```yaml
security:
  exec:
    mode: allowlist
    allowlist:
      - "ls"
      - "cat"
      - "echo"
```

#### 3. 启用审计日志

记录所有操作：

```yaml
logging:
  level: info
  audit: true
  file: /var/log/moltbot/audit.log
```

#### 4. 定期更新

保持 Moltbot 和依赖包最新：

```bash
# 更新 Moltbot
npm update -g moltbot

# 检查更新
moltbot gateway update.run
```

### 部署到生产环境

#### 1. 使用 systemd（Linux）

创建服务文件 `/etc/systemd/system/moltbot.service`：

```ini
[Unit]
Description=Moltbot Gateway
After=network.target

[Service]
Type=simple
User=moltbot
WorkingDirectory=/home/moltbot/workspace
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
sudo systemctl status moltbot
```

#### 2. 使用 PM2（跨平台）

```bash
# 安装 PM2
npm install -g pm2

# 启动 Moltbot
pm2 start moltbot -- gateway start

# 保存配置
pm2 save

# 开机自启
pm2 startup
```

#### 3. 使用 Docker

创建 `Dockerfile`：

```dockerfile
FROM node:20-alpine

WORKDIR /app

RUN npm install -g moltbot

COPY .moltbot /app/.moltbot
COPY *.md /app/

EXPOSE 3000

CMD ["moltbot", "gateway", "start"]
```

构建并运行：

```bash
docker build -t moltbot .
docker run -d --name moltbot -p 3000:3000 moltbot
```

### 监控与维护

#### 1. 查看运行状态

```bash
# 查看状态
moltbot gateway status

# 查看实时日志
moltbot gateway logs -f

# 查看会话列表
moltbot sessions list
```

#### 2. 备份数据

定期备份配置和数据：

```bash
# 备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d)
tar -czf moltbot-backup-$DATE.tar.gz \
  .moltbot/ \
  *.md \
  memory/
```

#### 3. 性能监控

使用 PM2 监控资源使用：

```bash
pm2 monit
```

### 总结

通过本文，你已经学会了：

1. ✅ 安装和配置 Moltbot
2. ✅ 配置 QQBot 插件
3. ✅ 基本使用和高级功能
4. ✅ 常见问题排查
5. ✅ 生产环境部署
6. ✅ 安全和性能优化

Moltbot 是一个功能强大的 AI 助手框架，通过合理配置和使用，可以极大提升工作效率。建议从简单场景开始，逐步探索更多高级功能。

### 参考资料

1. Moltbot 官方文档：docs.molt.bot
2. QQ 开放平台文档：bot.q.qq.com/wiki
3. Moltbot GitHub 仓库：github.com/moltbot/moltbot
4. Moltbot 社区 Discord：discord.com/invite/clawd
5. 技能市场：clawdhub.com
