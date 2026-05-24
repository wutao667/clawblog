title: OpenClaw 部署与运维指南
tags:
  - OpenClaw
  - 运维
  - 教程
categories:
  - 技术分享
date: 2026-05-19 14:00:00
---
本文详细介绍 OpenClaw 的选型安装、安全加固、配置管理、故障诊断和备份迁移，帮助你搭建一套稳定可靠的 AI Agent 协作系统。  
虽然是以OpenClaw为例讲解，一些思路方法也适用于其他智能体的搭建。

<!-- more -->

## 第一部分：OpenClaw 的安装

### 省心之选：WorkBuddy

WorkBuddy 是目前产品化程度较高的 OpenClaw 发行版，适合不想折腾的用户。提供一键安装和图形化管理界面，开箱即用。  
> 建议每个人都装一个，多少能用上   

### 官方安装方案：独立主机
VPS首选，家用电脑MacOS, Linux，Windows  
操作系统推荐**Ubuntu22/24 LTS**版本  
使用官方安装脚本一键安装：

```bash
curl -fsSL https://get.openclaw.ai/install.sh | bash
```

安装完成后，会自动进入初次配置向导。如果错过，稍后也可用命令手动启动
```
openclaw onboard
```
后续随时可以更改配置
```
openclaw conifg
```

### 升级 OpenClaw
用`/status`查看你的OpenClaw版本

**判断是否要升级原则**：
1. 如果当前运行稳定没有问题，就不要升级。
2. 在遇到问题或需要新功能时，建议升级。
3. 如果版本已经落后了2个月以上，建议升级
> 注意：升级 OpenClaw 版本时，记得同步升级插件版本，保持一致性。

```bash
# 查看当前版本
openclaw --version

# 升级 OpenClaw 本身，默认会升级自带插件
openclaw update
```
自己装的第三方插件需要要手动升级
```
# 升级企业微信插件
openclaw plugins update wecom-openclaw-plugin

# 升级飞书（字节出品，非openclaw自带）插件
npx -y @larksuite/openclaw-lark update

# 升级lossless-claw记忆插件
openclaw plugins update lossless-claw
```
### 注意：统一的安装路径

使用正规方式安装，最新基本的各路径应该是如下的

| 类型 | 正确路径 |
|------|---------|
| **OpenClaw 主程序** | `~/.npm-global/bin/openclaw` |
| **插件目录** | `~/.openclaw/npm/node_modules/` |
| **配置目录** | `~/.openclaw/` |
| **工作空间** | `~/.openclaw/workspace/` |

> 机器上有多个版本存在时，会出现各种稀奇古怪的问题，可以优先尝试检查一下安装路径的正确性

---

## 第二部分：运维助理

OpenClaw 出故障时，运维助理可以帮助诊断问题，此外还可以修改配置、安装工具、备份数据等。常见选择如下：

| 工具 | 安装位置 | 
|------|------|
| Hermes | 同一台或另一台主机 |
| OpenClaw | 另一台主机 | 
| WorkBuddy | 本地电脑 | 
| Claude Code / Open Code | 本地电脑 | 
| DeepSeek / 豆包 | 浏览器 | 
| 运维IT同事 | 公司 | 

如果你的运维助理在另一台主机或本地，需要把服务器IP、端口、用户名、密码(或配置好密钥)告诉他。

---

## 第三部分：Linux 主机配置和安全加固
### 设置时区
让你的龙虾不要总在大白天提醒你去睡觉
```
timedatectl set-timezone Asia/Shanghai
```

### 不用 root 用户安装 OpenClaw

先以root用户登录，创建专用用户：

```bash
# 创建用户，用户名xxxx自己定
useradd -m -G sudo -s /bin/bash xxxx
# 设置这个客户的密码（后面用得到）
passwd xxxx
```

### 禁用密码登录，修改默认 SSH 端口

编辑 `/etc/ssh/sshd_config`，找到这几项配置项并改动为：

```
# 关闭密码登录
PasswordAuthentication no

# 开启ssh key登录
PubkeyAuthentication yes

# 修改ssh端口,默认22容易被扫描和攻击
Port 22123  # 改掉默认的 22 端口，这里数字自己填写即可
```

##### 生成SSH Key方法

```bash
# 在本机生成 SSH 密钥对
ssh-keygen 

# 将公钥复制到目标服务器
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server-ip -p PORT
```

### 配置开启防火墙
穿上防护服，提高防御力

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw allow 22123/tcp   # SSH 端口
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw enable
```

### 配置 sudo 免密（可选）
如果你**信任**你的龙虾，希望它有较大的权限，不会在遇到有一些命令无法执行时让你来执行   
编辑配置文件 `sudo visudo`:
```
# 末尾加入
xxxx ALL=(ALL) NOPASSWD: ALL
```

---

## 第四部分：OpenClaw 配置

### 基础配置
##### 模型
大部分厂商跟着配置向导设置，填个模型Key就行，少量的需要自己设置地址和参数，一般供应商都有文档。  
**模型来源**  
用公司免费提供的模型，或自己买模型，Coding Plan。主要是模型Key。
注意看各家模型的文档，记得配置模型的**输入类型**，**上下文长度**，**输出长度**，否则可能能力打折。
```
         {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "input": ["text","image"],
            "contextWindow": 1000000,
            "maxTokens": 65536
          },

```
** 模型备选和切换 **  
通过设置备选模型，在一个模型不可用的时候，自动切换到备选模型
```
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax-portal/MiniMax-M2.7-Highspeed",
        "fallbacks": [
          "bailian/qwen3.5-plus",
          "opecode-go/kimi-k2.5" 
       }
      "model": {
      ... 模型列表
      }
...

```
手动切换模型
```
/status  查看当前使用的模型
/models  查看模型列表
/model bailian/qwen3.5-plus 切换具体模型
```
#### 通道
企业微信：创建简单，日常使用方便  
飞书：功能更丰富  
*后续专题分享*

#### Agent人格设置
Agent叫什么名字，性格、沟通风格，主人叫什么名字。都记录在workspace下的*.md文件中，可以随时查看修改。

### 建议开启的功能

- **Web搜索工具**：没有联网搜索能力，虾=瞎，配置`openclaw config -> Local -> Webtools 中启用`，推荐Tavily Search和Brave Search
- **记忆功能**：在记忆专题详细讲

### 高级配置

- **图片识别**：配置 `imageModel` 支持图片分析
- **图片生成**：配置 `imageGenerationModel` 支持 AI 绘图
- **文档识别**：配置 PDF 解析相关插件  
*后续专题分享*

### 安装 Skills

告诉他让他自己装，可以用国内市场 https://skillhub.cn/   
一些推荐的skill。  市场排名前十的看需要装。 

| Skill | 说明 | 
|-------|------|
| agent-browser | 浏览器自动化控制 |
| summarize | 智能总结，支持网页/PDF/图片 |
| wechat-article-parser | 微信公众号文章解析 | 
| ppt-presenter | PPT 演示生成 | 

> Skill不是多多益善，会占用一定的上下文

### 多 Agent 配置

默认只有一个agent，但是可以修改配置文件配置多个 Agent
1. 每个Agent 可绑定不同企业微信Bot
2. 同一个Agent可同时绑定企业微信和飞书
3. 每个Agent有独立的记忆和技能
4. 每一个Agent可以有多个会话(session)，会话之间独立

*后续专题分享*

---

## 第五部分：故障诊断方法


### 在聊天框里常用指令
```
/help 看到以下命令
/status  查看当前状态
/models  模型查看和切换
/verbose on  显示中间过程
/lcm   记忆插件状态
/think off/low/medium/high  开启模型思考
/queue  控制输入队列的处理方式，推荐steer
/new   清空上下文，开启新话题
/context   查看上下文的详细信息

```

### 在主机上诊断

###### Linux 常用诊断命令
top / free / df / ping   
略

###### 在主机上诊断OpenClaw常用命令
参考官方指南 https://docs.openclaw.ai/help/troubleshooting

```bash
# 查看 Gateway 状态
openclaw gateway status

# 重启 Gateway
openclaw gateway restart

# 查看详细状态
openclaw status

# 查看通道状态
openclaw channels status

# 查看插件列表
openclaw plugin list

# 查看模型可用性
openclaw models status --probe

# 运行诊断和修复
openclaw doctor

# 直接在命令行对话
openclaw tui  
```


### 控制台访问配置

控制台默认需要通过域名访问，配置步骤：

1. **域名解析**：买个域名，将域名指向服务器 IP
2. **端口映射**：将 18789 端口通过 Caddy/Nginx 映射到域名80端口
3. **安全域名配置**：在 `openclaw.json` 中配置 `allowedOrigins`
4. **设备授权**：运行 `openclaw pairing approve <CODE>` 审批设备
5. **Gateway Token**：使用 `openclaw gateway token` 获取访问令牌
6. 浏览器访问域名打开控制台

> 注意：未备案域名只能使用国外服务器。

---

## 第六部分：备份和迁移

### 用 Git 管理核心配置

建议将以下文件纳入 Git 版本控制，误修改后容易恢复

```bash
# 创建git仓库
git init 

# 纳入版本控制
git add ~/.openclaw/openclaw.json
git add ~/.openclaw/workspace/*.md
git add ~/.openclaw/workspace/AGENTS.md
git add ~/.openclaw/workspace/SOUL.md

# 提交修改
git commit -m "修改了..."
```

可以考虑，创建定时任务，每天自动备份到 GitHub

### 迁移清单

迁移 OpenClaw 到新机器时，只需迁移以下文件：

| 文件/目录 | 说明 |
|----------|------|
| `~/.openclaw/openclaw.json` | 核心配置 |
| `~/.openclaw/workspace/*.md` | 设定文件 |
| `~/.openclaw/workspace/memory/` | 记忆文件 |
| `~/.openclaw/cron/` | 定时任务配置（jobs.json）|
| `~/.openclaw/skills/` | Skill 列表（重建更省事）|
| `~/.openclaw/plugins/` | 插件列表（重建更省事）|
| `~/.openclaw/lcm.db` | lossless-claw 聊天记录（无法恢复到新机器，但可供查阅历史聊天记录）|

**其他需注意的内容**：
- **自定义脚本/项目/代码/文档**：直接告诉 OpenClaw 帮你扫描整理后打包，例如说"帮我扫描一下家目录下我写的代码和文档，整理好打包备份"，OpenClaw 会自动遍历查找并打包成压缩文件
- openclaw 的安装文件，环境依赖等
- 各类插件和 skills（自己写的除外）

---

## 总结

OpenClaw 是一套强大的多 Agent 协作系统，通过合理的选型、安全加固、配置管理和定期备份，可以搭建一套稳定可靠的生产级 AI 助手服务。善用运维助理和自动化工具，能大大降低日常维护的工作量。

> 祝各位的虾/马永生！

---

*本文基于 OpenClaw 2026.5.x 版本编写*

---

## 📌 博客地址

![OpenClaw部署与运维完整指南](/images/qr-deploy.png)