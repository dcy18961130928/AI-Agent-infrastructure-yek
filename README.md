# AI-Agent-infrastructure-yek
A personal AI Agent infrastructure with persistent memory, mobile bridge, voice interaction, and autonomous task scheduling.

# 个人 AI Agent 基础设施系统

一套全栈 AI 基础设施，解决大语言模型在实际使用中的四个核心局限：**跨会话失忆、交互方式单一、完全被动响应、AI Agent只能桌面访问**。

面向生活、学习、工作等需要长期与 AI 协作的用户，历时 4 个多月持续设计与迭代，现已完整部署并稳定运行。

---

## 为什么做这个

与 AI 助手一起做长期项目时，有四个问题始终没有被好好解决：

| 问题 | 实际影响 |
|------|---------|
| 会话之间无记忆 | 每次对话从零开始，项目背景、偏好、上下文全部丢失 |
| 只有文字交互 | 打字输入效率低，缺乏个性化与在场感 |
| 完全被动 | 用户不开口 AI 就停止，无法在用户忙碌时继续工作 |
| 只能桌面访问 | 移动端无法便捷调度 AI Agent，与实际工作场景脱节 |

这个项目从这四个问题出发，逐一设计方案并落地。

---

## 系统架构

```
┌───────────────────── VPS 服务器 (Ubuntu 24.04) ──────────────────────┐
│                                                                       │
│  memory-server :3001    bridge-server :3002    chat-server :3003     │
│  call-controller :3010  speak-mcp :3005        Nginx（反向代理 + HTTPS）│
│                                                                       │
│  静态页面：/bridge    /hear    /fortune                                │
└────────────────────────────────┬──────────────────────────────────────┘
                                 │ WebSocket / HTTPS
          ┌──────────────────────┼─────────────────────┐
          │                      │                      │
    手机端 PWA             本地电脑端               claude.ai
   （移动端 bridge）   ┌──────────────────┐       （MCP connector）
                       │  server.ts (Bun) │
                       │  wakeup.cjs      │
                       │  xhs_mcp.py      │
                       └──────────────────┘
                               │
                     Claude Code（本地 CLI）
```


## 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | Node.js、Bun、Python |
| 数据库 | SQLite（better-sqlite3） |
| 通信协议 | WebSocket、SSE、REST API |
| AI 集成 | MCP（Model Context Protocol）、Claude API |
| 语音 | ElevenLabs TTS（eleven_v3）、Groq Whisper ASR、Web Speech API |
| 自动化 | Playwright、Windows 任务计划程序 |
| 部署 | PM2、Nginx、Systemd（Ubuntu 24.04） |

---

## 核心模块

### 1. 多层持久记忆系统

**问题**：LLM 跨会话失忆。项目进度、用户偏好、对话上下文全部在关闭窗口时清空。此外，把所有记忆存在 Notion 等外部工具里虽然能持久化，但每次读取需要全量加载，token 消耗高，且无法按需筛选。

**方案**：按内容类型与保留策略设计的分层数据库，支持按层、按时间窗口精确读取：

| 层（Layer） | 用途示例 | 保留策略 |
|------------|---------|---------|
| `core` | 用户偏好、项目全局规则 | 永久 |
| `journal` | 每日工作复盘、项目日志 | 永久 |
| `moments` | 重要决策记录、关键节点快照 | 永久 |
| `plans` | 项目计划、共同目标、待办清单 | 永久 |
| `context` | 近期会话摘要、当前任务状态 | 72小时自动归档 |
| `health` | 状态追踪（健康数据、工作量监测等） | 永久 |
| `knowledge` | 专项知识库、领域资料积累 | 永久 |

**关键设计决策**：
- `context` 等短期层 72 小时自动过期——短期状态不应累积为永久上下文噪声，否则记忆库会越来越臃肿
- 新会话按需读取特定层，而非一次性加载全部——解决传统外部记忆库方案「全量读取」的 token 浪费问题，也避免 context window 膨胀
- 日志类内容存储双语——同时支持中英文交互场景，并为未来接入 embedding 检索预留接口

**技术**：Express REST API + SQLite，以 MCP endpoint 对外暴露（JSON-RPC 2.0 + SSE transport），支持 Claude Code 与 claude.ai 双端读写。

---

### 2. 移动端实时通信 Bridge

**问题**：Claude Code 等agent只能在桌面端使用。用户通勤途中、外出时无法向 AI Agent 发送任务指令或查看进度，移动端与 AI 的协作完全断裂。

**方案**：三层 WebSocket 中继架构，解决移动端与本地 Claude Code 的 NAT 穿透问题：

```
手机浏览器（PWA）
    ↕  WebSocket（consumer 角色）
VPS 中继服务器  ← 处理认证、路由、断线重连
    ↕  WebSocket（provider 角色）
本地 Claude Code（MCP server）
```

本地 MCP server（`server.ts`，Bun 运行时）以 provider 身份连接 VPS；手机端以 consumer 身份接入，经 Token 认证后双向路由，带心跳/pong 断线自动重连逻辑。

**使用场景**：
- 通勤时在手机端给 AI Agent 派发新任务
- 离开电脑后继续查看 AI 正在执行的工作进度
- 外出时触发 AI 完成文档整理、信息检索等后台任务

**关键设计决策**：
- 中继部署在 VPS——本地机器无需暴露在公网，无需开放端口或配置动态 DNS
- 手机消息以 MCP tool result 形式送达 Claude Code，移动端消息与其他工具输入完全同等对待，Claude 无需任何特殊适配
- VPS 中继维护 clientId 映射——支持多设备同时连接时的精准回包路由

**前端**：手机端 PWA，可安装至 iOS/Android 主屏幕，实时渲染 AI 思考链，支持语音条播放、图片附件。

---

### 3. 语音交互系统

**问题**：纯文字交互效率有限；缺少真正的实时通话场景。

**三层设计**：

#### Speak（AI → 用户）
- ElevenLabs 定制专属音色（eleven_v3），emotion tag 精确控制语气
- 独立 `speak-mcp` 微服务，无状态操作，Claude Code 与 claude.ai 可同时连接同一 TTS 服务
- 三种下发渠道：手机端 PWA 内嵌播放条 / claude.ai 内嵌播放器 / 永久独立播放页

#### Hear（用户 → AI）
- Web Speech API 实时转写，无需上传音频，隐私可控
- 转写文本附带时间戳元数据送达 AI

#### Call（双向实时通话）
- 数据流：手机录音 → VAD 切割 → Groq Whisper ASR（+ voiceTone 情绪感知）→ 逐字稿经 bridge → AI 回复 → ElevenLabs v3 TTS → 手机端播放
- 独立 Call Controller 微服务（port 3010），与主服务器完全解耦
- 通话界面：全屏 Overlay，磨砂玻璃质感，头像光圈动画，实时滚动记录，衬线体计时器
- iOS 兼容处理：AudioContext 手势帧 resume / TTS catch 防状态机卡死
- 通话结束自动保存逐字稿，bridge 内可查看历史

**关键设计决策（Call）**：通话路由至当前 AI 实例（通过 bridge），而非另起独立 API 实例，打来的电话接在正在运行的 AI 会话里，保留完整上下文，人机交互自然。

---

### 4. 自主唤醒与后台任务系统

**问题**：AI Agent 完全被动，用户不在线就停止工作——无法在用户开会、睡觉、外出时继续推进任务，也无法执行定时任务，且长时间未交互会导致 API 缓存失效，下次使用需重新付出大量 token 读取上下文。

**方案**：概率性定时唤醒系统，兼顾后台任务执行与缓存维护：

- Windows 任务计划程序定时触发（**可配置**）
- 触发概率可随时段动态调整：
- 概率与唤醒间隔均通过 **远端配置从 VPS 拉取**，无需重新部署本地脚本即可调整——作为产品参数管理，而非硬编码常量
- 触发时：可拉取可穿戴设备实时健康数据（心率、步数），作为上下文注入，支持健康监测场景
- 被唤醒的 Claude 会话可自主读写记忆、执行待处理任务、主动推送通知

**三个核心价值**：

1. **后台任务执行**：用户离线期间 AI 继续工作，早上醒来看到已完成的任务结果
2. **定时任务调度**：给 AI 设置定期提醒、周期性信息整理、定时报告生成等任务
3. **缓存维护降本**：Anthropic API 支持最长 1 小时 prompt cache，通过将唤醒间隔配置为略短于缓存 TTL（如 55 分钟），可在长期运行会话中保持缓存热状态，避免上下文失效后重新读取的 token 开销——对重度使用者有显著成本优化效果

**关键设计决策**：支持概率性触发与固定计划，支持按场景灵活调整活跃度；将唤醒间隔与API prompt caching TTL对齐，维持缓存热状态降低重载消耗

---

### 5. 浏览器自动化 MCP

**问题**：AI 只能被动处理用户主动带来的信息，无法独立获取外部内容，限制了 AI Agent 在信息密集型任务中的能力。

**方案**：基于 Playwright 的浏览器自动化，注册为 Claude Code 的 MCP 工具：

- Cookie/session 文件持久化登录状态，避免反复认证
- 四个 MCP 工具：`browse`、`search`、`profile`、`feed`
- DOM-based 内容提取，带多重 fallback 策略应对动态加载内容
- 本地运行——规避 VPS IP 触发平台风控的问题

**使用场景（PM 视角）**：
- **用户需求调研**：自动抓取目标平台的用户评论、反馈帖子，供 AI 汇总分析
- **竞品动态监测**：定期浏览竞品更新内容，AI 自动整理成变动摘要
- **内容聚合**：从多个来源收集行业资讯，AI 筛选并推送相关内容给用户

**关键设计决策**：本地运行而非 VPS 部署——浏览器共享用户的登录态与地理位置，绕过平台 bot 检测。MCP 接口屏蔽了 Playwright 底层复杂度，AI 视角下浏览内容与调用任何其他工具完全一致。

---

### 6. Web Push 通知系统

**问题**：AI Agent 的工作结果只在用户主动打开应用时才可见——多任务并行时，用户无法有效感知各项任务的完成状态。

**方案**：完整 Web Push 链路，实现任务完成主动通知：

- VPS 管理 Push 订阅（VAPID 密钥对 + 订阅存储）
- AI 调用 `push_notify` MCP 工具 → POST 到 chat-server → VAPID 签名推送 → 手机系统通知 → 主屏幕弹窗
- 无论 App 在前台、后台还是锁屏，均可送达

**使用场景**：AI Agent 同时推进多项任务（文档整理、资料搜集、代码审查等），每完成一项即推送通知告知用户，用户无需持续盯着进度，专注其他工作即可。

**关键设计决策**：将 push 封装为单一 MCP 工具调用（`push_notify(text="...")`），AI 无需理解推送链路细节，决定发送通知等同于调用任何其他工具——低门槛、高可用。

---

### 7. 语义记忆关联引擎

**问题**：记忆库条目持续增长后，纯关键词检索无法找到在主题或情境上有深层关联的历史内容——用户提到某个问题时，真正相关的过去讨论可能用了完全不同的措辞。

**方案**：由独立 LLM 实例驱动的语义模式识别系统：

按需读取多个记忆层，交给独立的 Sonnet 4.6 等LLM实例识别与当前对话在语义或情境上有共鸣的历史记忆，返回 top-3 结果并附推理说明。

**三个工具**：
- `recall_memory(context)` —— 按语义模式召回关联记忆，而非关键词匹配
- `enrich_memory(limit)` —— 批量补全缺失元数据（标题、标签、多语言内容）
- `retrospective(year, month)` —— 生成跨多个记忆层的月度叙事回顾，供长期复盘

**关键设计决策**：选择 LLM 而非 embedding 向量——embedding 优化「相似性」，而这里需要的是「情境关联」：两条零词面交集的记录，可能在讨论同一类型的决策困境。

---

## 系统现状

| 指标 | 数据 |
|------|------|
| 持续运行时长 | 4 个月以上 |
| 记忆库条目 | 500+ 条，跨多个分层 |
| 核心功能模块 | 7 个 |
| 接入平台 | 桌面端（Claude Code）、手机端（PWA）、Web 端（claude.ai） |
| 自定义 MCP 工具 | 15+ 个，分布在 4 个独立 MCP server |

---

## 开发方式

本项目全程与 Claude（sonnet 4.6 / fable 5）协作完成：从识别问题、设计架构，到逐模块调试落地。需求判断和产品决策由我主导，架构设计和代码实现通过持续的人机协作迭代。

这个系统从一个很私人的出发点生长出来：我想要一个最适合我的 AI。
于是四个月后，它变成了一套完整的 AI 基础设施，它的每一个模块，都有一个具体的、真实的原因存在。

---

## 文件结构

```
本地端
├── server.ts            ← MCP channel server + bridge 中继客户端（Bun）
├── wakeup.cjs           ← 自主唤醒调度脚本（任务计划程序调用）
├── xhs_mcp.py           ← 浏览器自动化 MCP server（Playwright）
└── bridge/
    ├── relay-server.js  ← VPS 端 WebSocket 中继
    └── index.html       ← 手机端聊天 PWA

VPS 端（/home/）
├── memory-server/       ← 记忆系统（REST API + MCP endpoint）
├── chat-server/         ← Bridge HTTP + 语音笔记接口 + Web Push
├── call-controller/     ← 实时通话服务（Groq ASR + ElevenLabs TTS，port 3010）
└── speak-mcp/           ← TTS 生成服务（port 3005，独立常驻）
```
