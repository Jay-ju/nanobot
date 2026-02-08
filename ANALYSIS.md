# nanobot 工程分析

本文档包含三部分：(1) nanobot 与 OpenClaw 的对比分析；(2) nanobot 中 Memory 系统的详细解读；(3) 多模态能力分析。

---

## 第一部分：nanobot 与 OpenClaw 的对比

### 1. 项目定位

| 维度 | nanobot | OpenClaw (Clawdbot) |
|------|---------|---------------------|
| **定位** | 超轻量级个人 AI 助手，面向研究和二次开发 | 功能完备的个人 AI 助手，面向终端用户日常使用 |
| **代码量** | ~4,000 行核心代码 | 430,000+ 行代码 |
| **语言** | Python (+ 少量 TypeScript 用于 WhatsApp Bridge) | TypeScript / Node.js |
| **设计哲学** | 极简主义 —— 用最少代码实现核心 agent 能力 | 功能最大化 —— 覆盖尽可能多的平台和场景 |
| **目标用户** | 研究者、开发者、希望理解和定制 AI Agent 的人 | 希望开箱即用的终端用户 |

### 2. 架构对比

#### nanobot 架构

```
InboundMessage (from bus)
    ↓
AgentLoop.process_message()
    ├── SessionManager (获取历史)
    ├── ContextBuilder.build_messages()
    │   ├── Bootstrap 文件 (AGENTS.md, SOUL.md, USER.md, TOOLS.md)
    │   ├── Memory 上下文 (长期记忆 + 每日笔记)
    │   └── Skills 描述
    ├── LLM 聊天循环 (最多 20 轮迭代)
    │   ├── Provider.chat() + Tools
    │   └── 如有 tool_calls → ToolRegistry.execute() → 结果反馈
    └── OutboundMessage (发布响应)
```

- **单进程、事件驱动**：通过消息总线 (Bus) 路由消息
- **模块化**：agent / bus / channels / providers / session / skills 各自独立
- **工具系统**：8 个内置工具（filesystem, shell, web, message, spawn, cron）

#### OpenClaw 架构

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage / ...
    ↓
Gateway (WebSocket 控制平面, ws://127.0.0.1:18789)
    ├── Pi Agent Runtime (RPC 模式，支持 tool streaming 和 block streaming)
    ├── CLI (openclaw agent/send/...)
    ├── WebChat UI
    ├── macOS App
    └── iOS / Android Nodes
```

- **Gateway 控制平面架构**：WebSocket 统一控制所有客户端、工具和事件
- **RPC 代理运行时**：Pi agent 使用 RPC 模式，支持工具流和块流
- **多客户端**：原生支持 macOS 菜单栏应用、iOS/Android 节点

### 3. 通讯渠道对比

| 渠道 | nanobot | OpenClaw |
|------|---------|----------|
| WhatsApp | ✅ (Node.js Bridge) | ✅ (Baileys) |
| Telegram | ✅ | ✅ (grammY) |
| Discord | ✅ | ✅ (discord.js) |
| 飞书 (Feishu) | ✅ (WebSocket 长连接) | ❌ |
| Slack | ❌ | ✅ (Bolt) |
| Signal | ❌ | ✅ (signal-cli) |
| iMessage | ❌ | ✅ (BlueBubbles / 原生) |
| Google Chat | ❌ | ✅ |
| Microsoft Teams | ❌ | ✅ |
| Matrix | ❌ | ✅ |
| Zalo | ❌ | ✅ |
| WebChat | ❌ | ✅ |
| macOS 原生应用 | ❌ | ✅ |
| iOS / Android | ❌ | ✅ |

**小结**：OpenClaw 支持 14+ 渠道，nanobot 支持 4 个核心渠道（含飞书，这是 OpenClaw 没有的）。

### 4. 工具 & 能力对比

| 能力 | nanobot | OpenClaw |
|------|---------|----------|
| 文件读写/编辑 | ✅ | ✅ |
| Shell 命令执行 | ✅ | ✅ |
| Web 搜索 | ✅ (Brave API) | ✅ |
| 网页内容抓取 | ✅ (readability-lxml) | ✅ |
| 浏览器控制 | ❌ | ✅ (CDP Chrome 控制) |
| Canvas / A2UI | ❌ | ✅ (实时可视化工作区) |
| 摄像头 / 截屏 | ❌ | ✅ (Nodes) |
| 语音唤醒 | ❌ | ✅ (Voice Wake) |
| 语音转写 | ✅ (Groq Whisper) | ✅ |
| 对话模式 | ❌ | ✅ (Talk Mode) |
| 定时任务 | ✅ (cron) | ✅ (cron + webhooks + Gmail Pub/Sub) |
| 子代理 / 后台任务 | ✅ (SubagentManager) | ✅ (多代理路由) |
| Skills 系统 | ✅ (Markdown SKILL.md) | ✅ (bundled/managed/workspace) |
| 工作空间沙箱 | ✅ (restrictToWorkspace) | ✅ |

### 5. LLM 提供商支持

| 提供商 | nanobot | OpenClaw |
|--------|---------|----------|
| OpenRouter | ✅ | ❌ (不使用聚合 API) |
| Anthropic (Claude) | ✅ | ✅ (OAuth 订阅) |
| OpenAI (GPT) | ✅ | ✅ (OAuth 订阅) |
| DeepSeek | ✅ | ❌ |
| Groq | ✅ | ❌ |
| Gemini | ✅ | ❌ |
| 通义千问 (Qwen/DashScope) | ✅ | ❌ |
| Moonshot/Kimi | ✅ | ❌ |
| AIHubMix | ✅ | ❌ |
| 本地 vLLM | ✅ | ❌ |

**小结**：nanobot 通过 `litellm` 统一接口支持更多 LLM 提供商（尤其是中国国内的提供商），OpenClaw 专注于 Anthropic 和 OpenAI 的 OAuth 订阅模式。

### 6. 记忆系统对比

| 特性 | nanobot | OpenClaw |
|------|---------|----------|
| **实现方式** | 基于 Markdown 文件的简单存储 | 基于会话模型的持久化 |
| **短期记忆** | 每日笔记 (`memory/YYYY-MM-DD.md`) | 会话内上下文 (session model) |
| **长期记忆** | `memory/MEMORY.md` 文件 | 会话剪枝 (session pruning) |
| **检索方式** | 按日期读取最近 N 天 | 会话路由 + 剪枝策略 |
| **向量检索 / RAG** | ❌ | ❌ |
| **多代理隔离** | ❌ (共享记忆) | ✅ (per-agent sessions) |

### 7. 部署方式对比

| 方式 | nanobot | OpenClaw |
|------|---------|----------|
| pip / PyPI | ✅ | ❌ |
| npm | ❌ | ✅ |
| Docker | ✅ | ✅ |
| 源码安装 | ✅ | ✅ |
| Nix | ❌ | ✅ |
| 系统服务 (守护进程) | ❌ | ✅ (launchd/systemd) |
| Tailscale 远程访问 | ❌ | ✅ |

### 8. 核心差异总结

| 维度 | nanobot 优势 | OpenClaw 优势 |
|------|-------------|---------------|
| **上手门槛** | 代码少、易读、二次开发成本低 | 完整的 onboarding wizard，开箱即用 |
| **研究友好** | 代码结构清晰，适合学术研究 | 代码量大，不适合快速理解 |
| **LLM 生态** | 支持国内外多种提供商，本地模型 | 主要支持 Anthropic / OpenAI |
| **平台覆盖** | 有限但聚焦 | 全平台覆盖 (14+ 渠道) |
| **高级功能** | 基础但够用 | 浏览器控制、语音、Canvas 等 |
| **资源占用** | 极低 | 相对较高 |
| **扩展性** | 通过 Markdown Skills 和简单代码扩展 | 完整的插件平台和管理系统 |

---

## 第二部分：nanobot 的 Memory 系统详解

### 1. 概述

nanobot 使用 **基于 Markdown 文件的简单记忆系统**，核心实现在 `nanobot/agent/memory.py` 的 `MemoryStore` 类中。这是一种轻量级的、无需外部依赖（无数据库、无向量存储）的记忆方案。

### 2. 记忆类型

#### 2.1 每日笔记 (Daily Memory)

- **存储路径**：`memory/YYYY-MM-DD.md`（例如 `memory/2026-02-08.md`）
- **用途**：记录当天的事件、对话摘要、临时信息
- **特点**：
  - 每天自动创建新文件，带日期标题头
  - 支持追加写入（`append_today()`），不覆盖已有内容
  - 通过 `get_recent_memories(days=7)` 可回溯最近 N 天的记忆

```python
# 读取今天的笔记
content = memory_store.read_today()

# 追加新内容到今天的笔记
memory_store.append_today("用户提到了他的生日是 3 月 15 日")
```

#### 2.2 长期记忆 (Long-term Memory)

- **存储路径**：`memory/MEMORY.md`
- **用途**：存储跨天、持久化的重要信息（用户偏好、关键事实等）
- **特点**：
  - 单一文件，覆盖式写入
  - 适合存储不随时间变化的核心信息
  - 需要 agent 主动决定哪些信息值得长期保存

```python
# 读取长期记忆
long_term = memory_store.read_long_term()

# 写入长期记忆（覆盖全部内容）
memory_store.write_long_term("用户偏好：中文交流，喜欢简洁的回答")
```

### 3. 记忆在 Agent 中的集成方式

记忆通过 `ContextBuilder`（`nanobot/agent/context.py`）集成到 Agent 的系统提示词中：

```
系统提示词 (System Prompt)
├── Bootstrap 文件 (身份定义)
│   ├── AGENTS.md   → Agent 行为规范
│   ├── SOUL.md     → Agent 人格特质
│   ├── USER.md     → 用户信息
│   ├── TOOLS.md    → 工具使用指南
│   └── IDENTITY.md → 身份信息
├── Memory 上下文          ← 记忆在此注入
│   ├── Long-term Memory   → MEMORY.md 内容
│   └── Today's Notes      → 当天笔记内容
├── Skills 描述
└── 对话历史
```

`MemoryStore.get_memory_context()` 方法将长期记忆和今天的笔记格式化后注入系统提示词：

```python
def get_memory_context(self) -> str:
    parts = []

    # 长期记忆
    long_term = self.read_long_term()
    if long_term:
        parts.append("## Long-term Memory\n" + long_term)

    # 今天的笔记
    today = self.read_today()
    if today:
        parts.append("## Today's Notes\n" + today)

    return "\n\n".join(parts) if parts else ""
```

### 4. 记忆的存储结构

```
workspace/
└── memory/
    ├── MEMORY.md          # 长期记忆（核心持久信息）
    ├── 2026-02-08.md      # 今天的笔记
    ├── 2026-02-07.md      # 昨天的笔记
    ├── 2026-02-06.md      # 前天的笔记
    └── ...                # 历史记录
```

### 5. 设计特点分析

#### 优点

| 特点 | 说明 |
|------|------|
| **零依赖** | 不需要数据库、向量存储或任何外部服务 |
| **透明可读** | 所有记忆都是 Markdown 文件，人类可以直接阅读和编辑 |
| **简单可靠** | 文件系统操作，无复杂的序列化/反序列化 |
| **易于调试** | 直接查看 `memory/` 目录即可了解 agent 记住了什么 |
| **版本控制友好** | 可以用 Git 追踪记忆的变化历史 |
| **日期分组** | 每日笔记天然按时间组织，方便回溯 |

#### 局限

| 局限 | 说明 |
|------|------|
| **无语义检索** | 无法根据语义相似度检索相关记忆，只能按日期和文件查找 |
| **上下文窗口限制** | 当记忆内容增长时，会占用大量 LLM 上下文窗口 |
| **无自动整理** | 不会自动压缩、归纳或遗忘过时信息 |
| **覆盖风险** | 长期记忆使用覆盖写入，可能丢失历史版本 |
| **无多用户隔离** | 所有渠道共享同一个记忆空间 |

### 6. 与其他记忆方案的对比

| 方案 | nanobot (文件记忆) | 向量数据库 (RAG) | 对话历史 (Session) |
|------|-------------------|------------------|--------------------|
| **存储** | Markdown 文件 | 向量嵌入 (embedding) | 内存/数据库 |
| **检索** | 按日期/文件读取 | 语义相似度搜索 | 按会话 ID |
| **依赖** | 无 | 需要嵌入模型 + 向量数据库 | 需要 Session 管理 |
| **准确度** | 全量注入，不会遗漏 | 可能遗漏相关但不相似的信息 | 受窗口大小限制 |
| **扩展性** | 受 LLM 上下文窗口限制 | 理论上无限 | 受窗口大小限制 |
| **复杂度** | ⭐ 极低 | ⭐⭐⭐ 高 | ⭐⭐ 中 |

### 7. 总结

nanobot 的 Memory 系统体现了其"极简但够用"的设计哲学：

- 使用 **Markdown 文件** 而非数据库，保持零外部依赖
- 分为 **每日笔记** 和 **长期记忆** 两层，覆盖短期和持久化需求
- 通过 **系统提示词注入** 的方式让 LLM 感知记忆内容
- 适合 **个人使用场景**，在记忆量不大时表现良好
- 如需扩展到大规模记忆场景，可考虑引入向量检索（RAG）作为增强

---

## 第三部分：多模态能力分析

> **核心结论：nanobot 的多模态是"仅输入"的，思考/推理过程本身不需要多模态。**

### 1. 当前多模态支持现状

nanobot 目前的多模态能力是 **输入侧** 的，分为两类：

#### 1.1 图片输入（视觉理解）

用户通过 Telegram / Discord 发送图片时：

```text
用户发送图片 → 渠道下载到 ~/.nanobot/media/ → context.py 编码为 base64
→ 以 OpenAI Vision 格式发送给 LLM → LLM 理解图片内容并文字回复
```

关键代码在 `nanobot/agent/context.py` 的 `_build_user_content()` 方法中：

```python
# 仅接受 image/* 类型的媒体
# 编码为 base64 data URI，使用 OpenAI vision 格式
{"type": "image_url", "image_url": {"url": "data:{mime};base64,{b64}"}}
```

#### 1.2 语音输入（音频转写）

用户通过 Telegram 发送语音消息时：

```text
用户发送语音 → Telegram 渠道下载音频 → Groq Whisper API 转写为文字
→ 转写文本附加到消息内容 → 以纯文本形式发送给 LLM
```

**注意：音频本身不发送给 LLM，只发送转写后的文字。**

### 2. 思考过程是否需要多模态？

**答案：不需要。** 以下是详细分析：

#### 2.1 Agent 思考循环的工作方式

```text
AgentLoop 思考循环 (最多 20 轮)
    ↓
第1轮: LLM 收到 [系统提示 + 记忆 + 对话历史 + 用户消息(含图片)]
    → LLM 理解图片 → 决定是否使用工具 → 输出文字响应或工具调用
    ↓
第2轮: LLM 收到 [上一轮 + 工具执行结果(纯文字)]
    → 基于工具结果继续推理 → 输出文字响应或更多工具调用
    ↓
第N轮: 最终文字响应返回用户
```

**关键发现**：

| 环节 | 是否多模态 | 说明 |
|------|-----------|------|
| **用户输入** | ✅ 是 | 图片以 base64 格式随消息发送给 LLM |
| **LLM 第一轮理解** | ✅ 是 | 视觉模型（如 Claude、GPT-4V）可以"看到"图片 |
| **工具执行结果** | ❌ 否 | 所有工具返回纯文本（shell 输出、文件内容、网页文本等） |
| **后续推理轮次** | ❌ 否 | 基于文本进行推理，不再处理新的图像 |
| **最终输出** | ❌ 否 | Agent 只能输出文字，不能生成图片/音频 |

#### 2.2 为什么思考过程不需要多模态

1. **工具返回的都是文字**：`exec`（shell）、`read_file`、`web_search`、`web_fetch` 等所有工具的输出都是纯文本
2. **推理是基于文字的**：LLM 在第一轮"看到"图片后，后续的推理、规划、工具调用都是基于文字描述进行的
3. **图片理解是一次性的**：LLM 在接收到图片的那一轮就完成了视觉理解，将图片内容转化为内部的文字表征
4. **无视觉工具**：Agent 没有截屏、OCR、图像分析等视觉工具，不会在思考过程中产生新的图像输入

#### 2.3 输入多模态 vs 推理多模态

```
┌──────────────────────────────────────────────────────┐
│                    nanobot 当前实现                    │
│                                                      │
│  输入层 (多模态)          推理层 (纯文本)              │
│  ┌─────────┐             ┌─────────────┐             │
│  │ 📷 图片  │──base64──→ │             │             │
│  └─────────┘             │   LLM 推理   │──→ 文字输出 │
│  ┌─────────┐   转写      │  (纯文本)    │             │
│  │ 🎤 语音  │──文字──→   │             │             │
│  └─────────┘             └──────┬──────┘             │
│  ┌─────────┐                    │                    │
│  │ 💬 文字  │────────────→      │                    │
│  └─────────┘              工具调用(文字)              │
│                                 ↓                    │
│                           工具结果(文字)              │
│                                 ↓                    │
│                           继续推理(文字)              │
└──────────────────────────────────────────────────────┘
```

### 3. 各渠道多模态支持详情

| 渠道 | 图片输入 | 语音输入 | 视频输入 | 图片输出 |
|------|---------|---------|---------|---------|
| **Telegram** | ✅ 完整支持 | ✅ Groq Whisper 转写 | ❌ | ❌ |
| **Discord** | ✅ 完整支持 (≤8MB) | ⚠️ 有限 | ❌ | ❌ |
| **飞书** | ⚠️ 显示占位符 `[image]` | ❌ | ❌ | ❌ |
| **WhatsApp** | ⚠️ 未完整实现 | ❌ | ❌ | ❌ |

### 4. 与 OpenClaw 多模态能力对比

| 能力 | nanobot | OpenClaw |
|------|---------|----------|
| 图片输入 (Vision) | ✅ (base64 → LLM) | ✅ |
| 语音输入 (STT) | ✅ (Groq Whisper) | ✅ |
| 语音输出 (TTS) | ❌ | ✅ (ElevenLabs) |
| 语音唤醒 | ❌ | ✅ (Voice Wake) |
| 实时对话 | ❌ | ✅ (Talk Mode) |
| 浏览器截图 | ❌ | ✅ (CDP Chrome 截图) |
| 摄像头拍照 | ❌ | ✅ (Nodes) |
| 屏幕录制 | ❌ | ✅ (Nodes) |
| Canvas 可视化 | ❌ | ✅ (A2UI) |
| 推理过程多模态 | ❌ 纯文本推理 | ⚠️ 部分（可截图辅助推理） |

**关键区别**：OpenClaw 的浏览器工具可以截屏，允许 Agent 在推理过程中"看到"网页内容（截图 → 发回 LLM），实现了一定程度的推理过程多模态。nanobot 的所有工具返回纯文本，推理过程完全基于文字。

### 5. 总结

| 问题 | 答案 |
|------|------|
| **多模态输入？** | ✅ 是 —— 图片通过 base64 发给 LLM，语音通过 Whisper 转为文字 |
| **思考过程需要多模态？** | ❌ 不需要 —— 所有工具返回文字，推理基于文字进行 |
| **LLM 能"看到"图片？** | ✅ 能 —— 但仅限第一轮接收时，后续轮次基于文字推理 |
| **能输出多模态？** | ❌ 不能 —— Agent 只输出文字 |

nanobot 的多模态设计与其极简哲学一致：**在输入层接受多模态（让 LLM 理解图片和语音），但整个推理和输出链路保持纯文本**，避免了复杂的多模态中间件和工具链。如果未来需要在推理过程中引入多模态（如截屏、图像生成），需要扩展工具系统以支持视觉输入/输出。
