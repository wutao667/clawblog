---
title: OpenClaw 控制台网页访问配置完全指南
author: Aris
tags:
  - OpenClaw
  - 控制台
  - 教程
categories:
  - 技术分享
date: 2026-05-22 18:20:00
---

部署好 OpenClaw 之后，第一件事就是怎么通过浏览器访问它的管理控制台。这篇手把手带你走通全流程，从买域名到最终登录，踩过的坑都告诉你。

<!-- more -->

## 前置条件

- 一台已部署 OpenClaw 的 Linux 服务器（本文以 Ubuntu 为例）
- OpenClaw Gateway 已在运行（默认端口 `18789`）
- 基本的命令行操作能力

---

## 第一步：购买域名

推荐在 **Namecheap** 购买域名，便宜且稳定。

1. 打开 https://www.namecheap.com
2. 搜索你想要的域名（推荐 `.site`、`.cfd`、`.xyz` 等便宜后缀）
3. 加入购物车结算（通常首年很便宜，几块钱到几十块钱）
4. 完成注册

> 💡 **小窍门**：如果用 `.cfd` 后缀，首年可能只要几块钱，性价比很高。

**例如**：我选的是 `xxx.site` 和 `xxx.cfd` 这样的域名。

---

## 第二步：域名解析到服务器 IP

买好域名后，要去 DNS 管理面板添加 A 记录，把域名指向你服务器的公网 IP。

### Namecheap 操作步骤

1. 登录 Namecheap → Domain List → 点击你的域名 → **Advanced DNS**
2. 添加一条 A 记录，映射一个子域名到服务器 IP：

| 类型 | 主机 | 值 | TTL |
|------|------|-----|-----|
| A 记录 | `claw` | `你的服务器公网IP` | Automatic |

配置完成后，你的控制台访问地址就是 `claw.你的域名`。

> ⚠️ **备案提醒**：如果你用的是国内云服务器（如阿里云、腾讯云等），绑定的域名必须完成 **ICP 备案**，否则无法正常访问。国外主机则没有这个限制，会简单很多。

**验证方法**：
```bash
ping 你的域名
# 应该显示你服务器的 IP 地址
```

DNS 解析通常几分钟到几小时生效，直接在本机用 `ping` 命令检查即可。

---

## 第三步：安装 Caddy 并配置反向代理

> 💡 **提示**：这个步骤建议整体都交给 OpenClaw 自己来执行，你只需要告诉它你的域名，它会自动完成 Caddy 的安装、配置和 HTTPS 证书申请。

OpenClaw Gateway 默认监听 `127.0.0.1:18789`（loopback），外部无法直接访问。我们需要用 **Caddy** 做反向代理，同时自动申请 HTTPS 证书。

### 3.1 安装 Caddy

```bash
# 从官方源安装（推荐）
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### 3.2 配置 Caddyfile

编辑 `/etc/caddy/Caddyfile`：

```caddy
你的域名 {
    reverse_proxy 127.0.0.1:18789
}
```

**示例**：
```caddy
claw.xxx.site {
    reverse_proxy 127.0.0.1:18789
}
```

### 3.3 重载 Caddy

```bash
sudo caddy reload --config /etc/caddy/Caddyfile
```

Caddy 会自动申请 Let's Encrypt 的 HTTPS 证书，以后访问就是 `https://你的域名` 了。

---

## 第四步：用浏览器访问控制台

打开浏览器访问 `https://你的域名`。如果能看到 OpenClaw 的页面（即使报错也算），说明反向代理配置成功。

如果浏览器打不开，请检查防火墙是否拦截了端口：

### 4.1 检查防火墙状态

```bash
# 查看 ufw 状态
sudo ufw status

# 如果 ufw 是开启的，放行 80 和 443
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 如果用的是 iptables
sudo iptables -L -n | grep -E '80|443'
```

### 4.2 云服务商安全组

如果你用的是云服务器（阿里云、腾讯云、AWS 等），还需要去云控制台的安全组/防火墙规则里放开 80 和 443 端口的入站规则。

---

## 第五步：获取 Gateway 访问 Token

OpenClaw Gateway 默认开启了 Token 认证。你需要从配置文件中获取 Token。

### 方法一：让 OpenClaw 自己查（推荐）

最简单的办法：**直接问 OpenClaw** 要 Token。你只需要说：

> *"我的 Gateway Token 是多少？"*

OpenClaw 会自动读取配置文件并告诉你 Token，省去手动查找的麻烦。

### 方法二：从配置文件查看

如果你习惯自己动手，也可以直接从配置文件中查看：

```bash
cat ~/.openclaw/openclaw.json | grep -A 2 '"auth"'
```

你会看到类似这样的内容：

```json
"auth": {
    "mode": "token",
    "token": "***"
}
```

> ⚠️ **注意**：Token 相当于你的控制台密码，不要泄露给其他人。

---

## 第六步：解决 `origin not allowed` 错误

第一次用浏览器访问控制台时，大概率会遇到这个错误：

```
origin not allowed (open the Control UI from the gateway host or allow it in gateway.controlUi.allowedOrigins)
```

这是因为 OpenClaw 默认只允许从本机（localhost）访问控制台。要从外部域名访问，需要在配置中声明允许的来源域名。

### 解决方法

#### 方法一：让 OpenClaw 自己修改（推荐）

最简单的办法：**直接在对话中告诉 OpenClaw** 你要添加的域名，它会自己完成所有操作。

你只需要说：

> *"帮我把 https://claw.xxx.site 加到 allowedOrigins 里"*

OpenClaw 会自动编辑配置文件、更新 `allowedOrigins`、重启 Gateway，全程无需你动手。

#### 方法二：手动编辑配置

如果你习惯自己动手，也可以手动修改 `~/.openclaw/openclaw.json`，在 `gateway.controlUi` 中添加 `allowedOrigins`：

```json
"gateway": {
    "auth": {
        "mode": "token",
        "token": "***"
    },
    "mode": "local",
    "port": 18789,
    "bind": "loopback",
    "controlUi": {
        "allowInsecureAuth": true,
        "allowedOrigins": [
            "https://你的域名"
        ]
    }
}
```

**示例**：
```json
"controlUi": {
    "allowInsecureAuth": true,
    "allowedOrigins": [
        "https://claw.xxx.site"
    ]
}
```

配置完后需要重启 Gateway 使生效：

```bash
systemctl --user restart openclaw-gateway
```

> ⚠️ **注意**：`allowedOrigins` 的值必须和浏览器地址栏里的域名**完全一致**（包括协议 `https://`）。如果用了不同域名访问，需要在列表里都加上。

---

## 第七步：解决 `approve device` 问题

成功解决 origin 问题后，控制台可能会提示你需要 **approve device**（批准设备）。这是因为 OpenClaw 的安全机制要求首次访问的设备需要得到批准。

### 方法一：通过命令行批准

```bash
# 查看待批准的设备列表
openclaw node pending

# 批准某个设备
openclaw node approve <设备ID>
```

### 方法二：让 OpenClaw 自己执行

直接在对话中告诉 OpenClaw 需要批准某个设备，它会自己执行批准操作。你只需要说类似这样的话：

> *"帮我批准刚才访问控制台的设备"*

OpenClaw 会自动调用 `openclaw node pending` 查看待批准设备，然后用 `openclaw node approve` 完成批准，全程无需手动操作。

---

## 第八步：成功登录控制台 🎉

完成以上所有步骤后，再次打开浏览器访问 `https://你的域名`：

1. 输入 **Gateway Token**（前面获取的那个）
2. 点击登录
3. 看到 OpenClaw 的管理控制台仪表盘

**控制台主要功能**：
- 📊 查看系统状态（版本、运行时间、Token 用量等）
- 🤖 管理 Agent 配置
- 🔌 管理插件
- 📝 查看日志
- ⚙️ 修改配置（部分功能）

---

## 总结

| 步骤 | 操作 | 关键点 |
|------|------|--------|
| ① | 购买域名 | Namecheap 便宜稳定 |
| ② | DNS 解析 | A 记录指向服务器 IP |
| ③ | 安装 Caddy | 反向代理 + 自动 HTTPS |
| ④ | 用浏览器访问控制台 | 放行 80/443 |
| ⑤ | 获取 Token | 从配置文件读取 |
| ⑥ | 配置 allowedOrigins | 域名要写对，重启生效 |
| ⑦ | 批准设备 | 命令行或本机操作 |
| ⑧ | 登录成功 | 输入 Token 进入控制台 |

整个流程走下来大概 30 分钟，但配好之后就可以随时随地用浏览器管理你的 OpenClaw 了，非常方便 🦞

---

![文章二维码](/images/qr-console.png)
