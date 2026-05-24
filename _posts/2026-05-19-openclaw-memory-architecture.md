title: OpenClaw 记忆机制详解：三层记忆架构与长期记忆管理
tags:
  - OpenClaw
  - 记忆机制
  - 教程
categories:
  - 技术分享
date: 2026-05-19 15:00:00
---
## 第一部分：三层记忆架构
OpenClaw 通过将记忆写入磁盘的纯文本文件来实现持久化——模型本身没有隐藏状态，所有记忆都显式存储在文件中。本文详细介绍其三层记忆架构及各种高级记忆功能。  

OpenClaw 的记忆系统采用**三层模型**，分别是：

### 第一层：会话上下文（Session Context）

当前对话的缓冲区，包含本轮对话的所有消息。这是每次 API 请求都会发送的活跃数据，由 LCM（Long-term Context Manager）管理。当上下文窗口接近满时，会触发自动压缩（compaction）。

**关键机制：自动记忆冲刷**

当对话接近上下文上限时，OpenClaw 会触发一个静默的 Agent 自主执行，在压缩前自动将重要上下文写入 `memory/YYYY-MM-DD.md`：

```
对话接近上下文限制
    ↓
静默记忆冲刷触发
    ↓
模型将重要上下文写入 memory/ 日志
    ↓
模型回复 NO_REPLY（用户无感知）
    ↓
自动压缩继续执行
```

### 第二层：长期记忆（memory文件夹）

位于`workspace/memory/`下，可能包含 `memory/YYYY-MM-DD.md`等自动生成的文件，记录每日运行日志和观察，**也可以自己添加md文件进去**。今天和昨天的日志会自动加载，文件会被索引供 `memory_search` 检索，但不会注入到每次启动的引导提示中。

适用于：详细的每日笔记、观察、对话摘要和将来可能有用的原始上下文和任何md格式的资料。

### 第三层：核心记忆（MEMORY.md）

位于 `~/.openclaw/workspace/MEMORY.md`，是精心提炼的持久层。记录持久的facts、偏好、决策和摘要，在每次主会话启动时自动加载。

适用于：长期facts、用户偏好、既定决策和简短摘要。**不是**原始对话记录或每日日志。

---

三层之间的关系：

| 层级 | 文件 | 生命周期 | 何时加载 |
|------|------|---------|---------|
| 第一层 | 会话上下文 | 当前对话 | 每次对话发送 |
| 第二层 | memory/*.md | 每日/长期 | 今天+昨天自动加载，随时可以检索 |
| 第三层 | MEMORY.md | 长期驻留 | 每次 DM 会话启动 |

---

## 第二部分：核心记忆文件 MEMORY.md

### 什么是 MEMORY.md

`MEMORY.md` 是 OpenClaw 的**核心记忆文件**，位于工作空间根目录。每次新的主会话（DM）启动时，这个文件的内容会自动加载到模型上下文中，确保 Agent 能够"记住"之前的重要信息。

### 写入什么内容

- 用户的持久偏好（如"使用中文回复"）
- 已做出的重要决策和结论
- 长期的facts（如用户的姓名、职业、项目信息）
- 技能和工具的配置说明
- 协作规则和流程

### 不要写入的内容

- 原始对话记录（这是 daily notes 的职责）
- 临时性的待办（用飞书任务/企业微信待办）
- 过时信息（定期清理 MEMORY.md）

### 如何维护

- 让 Agent 自动维护即可。当你告诉它"记住 XXX"时，它会自动写入合适的位置。定期（建议每隔几天）可以让 Agent 审查 MEMORY.md，删除过时内容，将详细材料提炼为摘要。  
- MEMORY.md有长度限制，默认12000字节，超过会截取，太长时一般自动将详细内容移回 `memory/*.md`，或在 MEMORY.md 中只保留精简摘要。

> 当龙虾说："我记住了"，问问它"你记到哪里了，文件发我看看"。基本上能解决偷懒现象。  


---

## 第三部分：上下文管理

### 上下文中包含的内容

每次 API 请求发送的上下文包含：

- 当前对话的**历史消息**
- 引导提示（Bootstrap prompt）
- 已加载的 MEMORY.md 内容
- 已加载的 daily notes 内容
- 工具定义和调用历史
- 系统指令  
用 `/context list`可以查看

> **默认每天4:00会清空上下文，"睡一觉失忆了"  **  
/reset /new 也具有有类似效果    

解决方法：将自动会话重置条件改成空闲7天
```
  "session": {
    "dmScope": "per-channel-peer",
    reset: {
      mode: "idle",
      idleMinutes: 10080
    }
  },

```
`idle,10080`表示空闲7天后重置   
`per-channel-peer`表示会话按照通道+发送者的区分独立
#### 找回上下文
1. 方法一：在你的聊天软件记录中，手动复制发给它
2. 方法二：让它自己找，聊天记录完整版在.openclaw/agents/xxx/sessions/\*.jsonl 按照更新时间和大小筛选

### 默认引擎的处理方式

OpenClaw 默认使用 **LegacyContextEngine** 进行上下文管理。当对话变长、接近上下文窗口上限时，会触发**自动压缩（Auto-compaction）**：

1. 评估哪些消息可以安全压缩或移除
2. 保留关键信息和结论
3. 用摘要替代原始对话记录
4. 释放上下文空间继续对话


### 无损上下文：lossless-claw 插件

默认的压缩方式会丢失部分对话细节。如果你希望**无损保留所有对话历史**，可以安装 `lossless-claw` 插件：

```bash
openclaw plugins install @martian-engineering/lossless-claw
```
在对话框中输入 `/lcm`确认有没有生效，应该看到
```
enabled: yes
selected: yes (slot=lossless-claw)
...
```
如果没生效，检查配置文件是否包含
```
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    }
  }
}
```

**lossless-claw 的工作原理**：

- 将每次交互存储在持久化数据库中
- 将旧内容**增量摘要**为层级有向无环图（DAG）
- 不删除任何信息，只提炼关键结论
- Agent 可以动态重构详细上下文（按需展开摘要）
- 保留完整的对话历史，而非删除后不可恢复

> 注意：lossless-claw 不会写入 MEMORY.md，只管理会话压缩过程。

---

## 第四部分：长期记忆与记忆搜索
### 查看memory目录
检查 ~/.openclaw/workspace/memory 目录，这些是长期对话积累下来的核心记忆文件。  
如果配置妥当，这些记忆文件都是龙虾回忆和检索信息的素材。

### 开启长期记忆功能
memory-core是内置插件，负责读取和搜索记忆文件，提供 `memory_search` 和 `memory_get` 两个工具，启用方法
在 `openclaw.json` 中配置：
```json
{
  "plugins": {
   "memory-core": {
     "enabled": true
   }
}

```

### 配置记忆搜索和本地向量模型
> 向量模型是什么？
[直观演示](https://embedding-representations.vercel.app/)

memorySearch 是 memory-core 插件的行为配置
在 `openclaw.json` 中配置：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "local"
      }
    }
  }
}
```
其中 `provider: "local"` 使用本地向量模型进行语义搜索，无需外部 API。  
需要安装`node-llama-cpp`并下载本地向量模型  hf_ggml-org_embeddinggemma-300m-qat-Q8_0.gguf
```
npm install node-llama-cpp
cd ~/.node-llama-cpp
npx --no node-llama-cpp pull --dir ./models hf:ggml-org/embeddinggemma-300m-qat-Q8_0-GGUF
```

### 查看当前记忆状态

```bash
# 查看记忆文件大小和状态
openclaw memory status
```

### 建立索引数据库

首次配置后，手动使用向量模型对记忆库内容建立索引

```bash
# 索引记忆文件
openclaw memory index

# 重建所有记忆文件索引
openclaw memory index --force
```
后续监测memory文件夹，变动后自动更新索引
```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "local",
        "sync": {
          "watch": true,
          "watchDebounceMs": 1500
        }
      }
    }
  }
}
```

---

## 第五部分：Dreaming 做梦机制

### 什么是 Dreaming

Dreaming 是 OpenClaw v2026.4.5 引入的**实验性记忆巩固框架**，灵感来源于人类睡眠机制。Agent 在后台（类睡眠状态）自动整理、提炼和强化记忆，帮助自己在"醒来"时拥有更清晰可用的记忆系统。

### 工作原理

Dreaming 模拟人类睡眠的三个阶段：

| 阶段 | 类型 | 功能 |
|------|------|------|
| 浅睡 | 摄入（Ingestion） | 消化当日记忆文件 |
| REM | 模式识别 | 识别重要模式和关联 |
| 深睡 | 记忆强化 | 将重要内容提升到长期记忆 |

### Dreaming 的输出

Dreaming 会在工作空间生成 `DREAMS.md` 文件（如果存储包含内联输出），记录：
- REM 睡眠摘要
- 做梦期间的强化信号
- 用于深度排序的记录

> 注意：Dreaming **不会**写入 MEMORY.md，只写入 DREAMS.md 供人类审查。

### 开启 Dreaming

```json
{
  "plugins": {
   "memory-core": {
     "enabled": true,
     "config": {
       "dreaming": {
         "enabled": true,
         "frequency": "0 3 * * *",
         "timezone": "Asia/Shanghai"
       }
     }
   }
}

# 重启 Gateway
openclaw gateway restart
```

### 查看Dreaming 日志
Dreaming 运行后  
主要查看`.openclaw/workspace/memory/dreaming`目录内容   
也可以查看生成的 DREAMS.md（仅供娱乐）：

```bash
cat ~/.openclaw/workspace/DREAMS.md
```
也可以在控制台查看

---

## 第六部分：知识库

OpenClaw 支持多种知识库集成，实现超越纯文本记忆的结构化知识管理。

### 内置知识库：OpenClaw Wiki

OpenClaw自带的`memory-wiki` 插件，基于md文件打造 Wiki 知识库，[官方文档](https://docs.openclaw.ai/plugins/memory-wiki#dashboards-and-health-reports)。开启方法：在 openclaw.json 的 plugins.entries 部分设置

```bash
{
  "plugins": {
    "entries": {
      "memory-wiki": {
        "enabled": true
      }
    }
  }
}

```
知识库文档存放在`~/.openclaw/wiki/main/*.md`  
开启后Agent可用以下工具：
- `wiki_search` — 知识库搜索
- `wiki_get` — 读取知识页面
- `wiki_apply` — 写入知识更新
- `wiki_lint` — 检查知识库一致性
> 功能较弱，但是节省资源，适合轻量级的个人知识库

### 外部知识库：RAGFlow

[RAGFlow](https://ragflow.io) 是一个开源的 RAG（检索增强生成）引擎，可以与 OpenClaw 配合使用，实现更强大的文档理解和问答能力。  
RAGFlow 支持丰富的文件格式，涵盖了办公文档、表格、幻灯片、纯文本、图像、邮件、音频、视频、网页等多种类型。

适合场景：
- 大规模文档库检索
- 复杂文档的语义理解
- 企业知识管理

公司统一建立了RAGFlow [https://ragflow.zhgcloud.com](https://ragflow.zhgcloud.com), 联系运维开通账号，获取资料，贡献资料   
Confluence上有[使用指南](https://doc.zeaho.com/pages/viewpage.action?pageId=386408211)，包括Skill安装和配置教程。
> 广泛征集知识库的贡献者

### 外部知识库：Get笔记

Get笔记（biji.com）[开放平台](https://www.biji.com/openapi?tab=docs)，支持将笔记内容接入 OpenClaw：
```
clawhub install getnote
```
安装完成后说「请帮我授权 Get笔记」，AI 自动生成授权链接，点击完成登录，无需手动配置 API Key。

配置后可以通过 Skill 调用 Get笔记的读取、搜索和写入功能。

### 外部知识库：IMA 笔记
ima（全称 ima.copilot）是腾讯推出的一款以知识库为核心的 AI 智能工作台，可以通过其开放 Skill 与 OpenClaw 集成    
[官方ima skil](https://ima.qq.com/agent-interface)

```
请安装 ima 技能
下载地址：https://app-dl.ima.qq.com/skills/ima-skills-1.1.7.zip
API Key 获取：https://ima.qq.com/agent-interface
```
获取API Key，发给小龙虾以完成配置。   

---


## 第七部分：当 Agent "想不起来"和"不知道"时怎么办

有时候 Agent 会说"我想不起来了"，这时候可以引导它去各个记忆系统中搜索。

### 搜索途径

**搜索记忆文件**  
直接告诉它："回忆一下XXX"或"在记忆里找找看"。  
Agent 会使用 `memory_search` 工具进行语义搜索，即使用词不同也能匹配到相关内容。

**搜索之前的对话记录**  
告诉它："用 lcm_grep 搜索一下XXX"或"查一下seesion日志里最近聊的内容"。  
你要的会话历史一定在某个文件里存着。

**搜索外部知识库**  
如果知识库已配置，告诉它："去知识库里搜索一下"或"在 RAGFlow/Get笔记里找找XXX知识"。  

**上网搜索**  
告诉它："你去查查XXX" "去详细调研一下"

### 示例对话

> **用户**：你记得上周我们讨论的 OpenClaw 升级方案吗？  
> **Agent**：我想不起来了...  
> **用户**：去记忆里搜索一下 / 用lcm grep搜一下XX   
> **Agent**：`memory_search` 搜索中`lcm_grep`...找到了！上周我们讨论了... 

---

## 总结

OpenClaw 的记忆系统设计精妙，通过三层架构实现了"无隐藏状态"的纯文本持久化：

1. **会话上下文**：活跃对话缓冲区，由 LCM 管理
2. **长期记忆**：每日笔记，md文件资料库
3. **核心记忆**：精心提炼的 MEMORY.md

配合 **lossless-claw** 实现无损压缩、**Dreaming** 实现睡眠式记忆整理、以及多种**知识库**集成，构成了一个完整的长期记忆管理生态系统。

善用这些机制，可以让 AI Agent 真正做到"记得住、记得准"。  
> 祝各位的虾/马不忘初心！

---


*本文基于 OpenClaw 2026.5.x 版本编写*

---

## 📌 博客地址

![OpenClaw记忆机制详解](/images/qr-memory.png)