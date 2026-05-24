---
title: OpenClaw 的图片识别与生成能力配置指南
author: Aris
tags:
  - OpenClaw
  - 图片
  - 多模态
  - 教程
categories:
  - 技术分享
date: 2026-05-22 20:00:00
---

OpenClaw 不仅能处理文本，还能「看懂」图片和「画出」图片。这篇文章带你了解背后的模型能力，以及如何在 OpenClaw 中配置和使用这些功能。

<!-- more -->

---

## 一、多模态模型的能力概览

在深入配置之前，先搞清楚几个概念：

| 能力类型 | 说明 | OpenClaw 中的配置项 |
|---------|------|-------------------|
| **文本理解** | 处理文字输入、对话、推理 | 默认模型即可 |
| **图片识别** | 看懂图片内容、OCR 文字提取、物体识别 | `imageModel` |
| **图片生成** | 根据文字描述生成图片 | `imageGenerationModel` |
| **视频理解** | 分析视频内容（部分模型支持） | 取决于所选模型 |
| **视频生成** | 根据文字生成视频（需特定模型） | 插件或 API 调用 |
| **向量嵌入** | 将内容转为向量用于语义搜索 | `memorySearch` |

OpenClaw 的 `image` 和 `image_generate` 两个工具分别对应图片识别和图片生成能力，通过配置不同的模型来驱动。

---

## 二、主流多模态模型对比

### 国外模型

| 模型 | 图片识别 | 图片生成 | 视频理解 | 视频生成 | 特点 |
|------|:------:|:--------:|:--------:|:--------:|------|
| GPT-4o / GPT-4.1 | ✅ | ✅ | ✅ | ❌ | 多模态均衡，1M 上下文 |
| Gemini 2.5 Pro | ✅ | ✅ | ✅ | ❌ | 原生多模态，超长上下文 |
| Claude 4 / 3.5 Sonnet | ✅ | ❌ | ❌ | ❌ | 图片理解强，不支持生成 |

### 国内模型

| 模型 | 图片识别 | 图片生成 | 视频理解 | 视频生成 | 特点 |
|------|:------:|:--------:|:--------:|:--------:|------|
| **Qwen3.5-Plus** | ✅ | ❌ | ✅ | ❌ | 阿里原生多模态，图文理解强 |
| **Qwen3.6-Plus** | ✅ | ❌ | ✅ | ❌ | 增强 Agentic Coding，多模态输入 |
| **MiniMax-M2.7** | ❌ | ❌ | ❌ | ❌ | **纯文本模型**，专注 Coding 和 Agent，不支持多模态 |
| **MiniMax-M2.5** | ❌ | ❌ | ❌ | ❌ | 纯文本模型，同上 |
| **MiniMax image-01** | ❌ | ✅ | ❌ | ❌ | **专用图片生成模型** |
| **GLM-5 / GLM-4.7** | ✅ | ❌ | ✅ | ❌ | 智谱多模态，中文优化 |
| **Kimi K2.5 / K2.6** | ✅ | ❌ | ✅ | ❌ | 月之暗面，超长上下文 |

> 💡 **关键区别**：图片识别（看懂图片）和图片生成（画图）是两套不同的模型。通常一个模型只擅长其中一项，所以 OpenClaw 需要分别配置。

> ⚠️ **关于 MiniMax**：MiniMax-M2.7 / M2.5 **自身是纯文本模型**，不支持图片输入。但如果你通过 `minimax-portal` 提供商（OAuth 登录）使用 OpenClaw，它会自动启用 MiniMax 的 **Image Understanding MCP** 能力，实现图片识别。这是 OpenClaw 的插件机制提供的，跟 M2.7 模型本身无关。

### 以阿里为例

阿里百炼平台的通义千问系列中：
- **Qwen3.5-Plus / Qwen3.6-Plus**：支持图片、视频输入，具备强大的**图像识别和理解**能力，但不支持图片生成
- 如果需要图片生成，则需要搭配专门的图像生成模型

### 以 MiniMax 为例

MiniMax 系列中：
- **MiniMax-M2.7 / M2.5**：**纯文本模型**，不支持图片输入。但如果通过 `minimax-portal` OAuth 使用，OpenClaw 会自动启用 Image Understanding MCP 来提供图片识别能力
- **MiniMax image-01**：**专门的图片生成模型**，根据文字生成高质量图片，支持多种风格和比例

---

## 三、配置图片能力

OpenClaw 通过两个配置项分别控制图片识别和图片生成能力：

| 配置项 | 作用 | 推荐模型 |
|--------|------|---------|
| `imageModel` | 图片识别（看懂图片） | `bailian/qwen3.5-plus` |
| `imageGenerationModel` | 图片生成（画出图片） | `minimax-portal/image-01` |

### 配置路径示意

```
gateway
└── agents
    └── defaults
        ├── model              ← 对话模型
        ├── imageModel         ← 图片识别模型
        └── imageGenerationModel ← 图片生成模型
```

下面分别介绍两项配置的具体方法。

---

### 3.1 配置图片识别（imageModel）

查看当前配置：

```bash
cat ~/.openclaw/openclaw.json | grep -A 3 '"imageModel"'
```

如果没有输出，说明未配置。在 `agents.defaults` 中添加：

```json
"imageModel": {
    "primary": "bailian/qwen3.5-plus"
}
```

> 💡 **推荐模型**：`bailian/qwen3.5-plus` 是阿里百炼的通义千问多模态模型，图片理解能力出色，中文支持好。也可用 `bailian/qwen3.6-plus`（更新版）。

### 3.2 配置图片生成（imageGenerationModel）

查看当前配置：

```bash
cat ~/.openclaw/openclaw.json | grep -A 3 '"imageGenerationModel"'
```

在 `agents.defaults` 中添加：

```json
"imageGenerationModel": {
    "primary": "minimax-portal/image-01"
}
```

> 💡 **推荐模型**：`minimax-portal/image-01` 是 MiniMax 的专用图片生成模型，生成质量高，支持多种比例和风格。

### 3.3 让 OpenClaw 自己配置（推荐）

不想手动改文件？直接告诉 OpenClaw 就行：

> *"帮我配置图片识别模型，用 qwen3.5-plus"*
> *"帮我配置图片生成模型，用 minimax-portal/image-01"*

OpenClaw 会自动编辑配置文件并重启 Gateway。

### 3.4 完整配置示例

```json
"agents": {
    "defaults": {
        "model": {
            "primary": "opencode-go/deepseek-v4-flash",
            "fallbacks": [...]
        },
        "imageModel": {
            "primary": "bailian/qwen3.5-plus"
        },
        "imageGenerationModel": {
            "primary": "minimax-portal/image-01"
        }
    }
}
```



## 四、测试效果

### 4.1 测试图片识别

给 OpenClaw 发一张图片，让它描述内容：

> *"这张图里有什么？"*

OpenClaw 会调用 `image` 工具，将图片传给配置的 `imageModel` 进行分析并返回结果。

**示例效果**（我自己的测试）：
```
用户：这张图里有什么？
→ OpenClaw 调用 image 工具 → qwen3.5-plus 分析 → 返回描述
```

### 4.2 测试图片生成

让 OpenClaw 生成一张图片：

> *"帮我生成一张小狗的图片"*

OpenClaw 会调用 `image_generate` 工具，使用配置的 `imageGenerationModel` 生成图片。

**示例效果**（我自己的测试）：
```
用户：生成一只小狗的图片
→ OpenClaw 调用 image_generate → minimap-portal/image-01 生成 → 返回图片
```

### 4.3 发送生成的图片

图片生成完成后，OpenClaw 需要通过 **message 工具** 把图片发送给你。这是因为图片生成是后台任务，生成的图片需要主动投递到对话中。

流程如下：

```
你让 OpenClaw 生成图片
  → OpenClaw 调用 image_generate（后台任务）
  → 图片生成完成
  → OpenClaw 收到完成通知
  → 调用 message 工具将图片发送给你 ✅
```

---

## 五、常见问题

### Q: 配置了 imageModel 但图片识别不工作？
A: 确认模型是否支持图片输入。不是所有模型都支持多模态，例如 `opencode-go/deepseek-v4-flash` 就不支持图片识别，只能用文本来生成图片？不对，纯文本模型根本无法识别图片。所以一定要配一个支持多模态的模型。

### Q: 图片生成特别慢？
A: `image_generate` 是后台异步任务，生成时间取决于模型服务端的负载。通常 10-30 秒完成。

### Q: 生成的图片怎么没显示在对话里？
A: 检查 OpenClaw 的 message 工具权限是否正确。OpenClaw 需要通过 message 工具把图片发送回来。

---

## 总结

| 能力 | 配置项 | 推荐模型 | 主要工具 |
|------|--------|---------|---------|
| 图片识别 | `imageModel` | `bailian/qwen3.5-plus` | `image` |
| 图片生成 | `imageGenerationModel` | `minimax-portal/image-01` | `image_generate` |

记住两条核心原则：
1. **图片识别和图片生成是两套模型**，需要分别配置
2. **不想手动改配置？直接告诉 OpenClaw 就好** 🦞

---

![文章二维码](/images/qr-image.png)
