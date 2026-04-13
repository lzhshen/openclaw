# OpenClaw Gateway Chatbot 开发指南

本文面向应用开发者，说明如果你希望基于 OpenClaw Gateway 开发一个定制 Chatbot，应该优先复用哪些接口、推荐采用什么接入方式，以及会话管理接口分别意味着什么、能支撑哪些用户可感知功能。

---

## 1. 先做技术选型：WebSocket 还是 HTTP？

## 推荐结论

如果你的目标是：

- 尽量复用现有 OpenClaw Dashboard 的设计
- 需要会话列表、实时消息、运行状态、审批、Presence、历史同步
- 需要更“原生”的 OpenClaw 客户端体验

那么推荐优先采用：

### WebSocket Gateway Protocol

核心参考：

- `docs/gateway/protocol.md:10`
- `src/gateway/protocol/schema/frames.ts:20`
- `src/gateway/protocol/schema/protocol-schemas.ts:197`
- `ui/src/ui/gateway.ts:323`
- `ui/src/ui/app-gateway.ts:206`

如果你的目标只是：

- 快速接入聊天能力
- 兼容 OpenAI / Responses 风格生态
- 不强依赖 OpenClaw 的 session 与 event 模型

那么可以考虑：

### HTTP 接口
- `POST /v1/chat/completions`
- `POST /v1/responses`

核心参考：

- `docs/gateway/openai-http-api.md:8`
- `docs/gateway/openresponses-http-api.md:9`

---

## 2. 为什么模仿 Dashboard 时优先选 WebSocket？

当前 Dashboard / Control UI 的主通道并不是 HTTP + SSE，而是 WebSocket。

直接证据：

- `ui/src/ui/gateway.ts:327`：`new WebSocket(this.opts.url)`
- `ui/src/ui/gateway.ts:517`：发送 `connect`
- `ui/src/ui/gateway.ts:614`：通过 `ws.send(JSON.stringify(frame))` 发送 RPC frame
- `ui/src/ui/app-gateway.ts:301`：接收 Gateway event

Dashboard 的主工作方式是：

1. WebSocket 建连
2. `connect` 握手
3. 通过 `req/res` 方式调用 Gateway RPC
4. 通过 `event` 接收实时推送

所以，如果你想做“像 Dashboard 一样”的 Chatbot，最自然的方案是直接实现一个 Gateway WS client。

---

## 3. 推荐的最小实现方案

如果你要做一个最小可用、但依然比较原生的 OpenClaw Chatbot，建议先只实现下面这一小组接口。

## 必需接口

### 握手与连接
- `connect`

### 聊天
- `chat.send`
- `chat.history`
- `chat.abort`

### 会话
- `sessions.subscribe`
- `sessions.list`
- `sessions.reset`
- `sessions.patch`（可选，但强烈建议）

### 事件
- `chat`
- `session.message`
- `sessions.changed`

### 可选增强
- `agent.identity.get`
- `models.list`
- `system-presence`
- `health`

---

## 4. 一个推荐的最小交互流程

下面是一种比较适合定制 Chatbot 的流程。

### 启动阶段
1. 通过 WebSocket 连到 Gateway
2. 发送 `connect`
3. 调用 `sessions.subscribe`
4. 调用 `sessions.list`
5. 选择一个 sessionKey（或创建默认会话）
6. 调用 `chat.history` 拉历史

### 用户发消息
1. 客户端调用 `chat.send`
2. 记录 runId / sessionKey
3. 监听 `chat` event
4. 收到终态后，如果需要，重新调用 `chat.history`

### 用户切换会话
1. 调用 `sessions.list`
2. 选择目标 `sessionKey`
3. 调用 `chat.history`

### 用户点击“新对话”
1. 调用 `sessions.reset` 或 `sessions.create`
2. 切换到新 sessionKey
3. 清空当前展示并等待新消息

---

## 5. 会话管理接口详解：含义与可开发的用户功能

这一节专门解释此前提到的“C. 会话管理相关接口”。

---

### 5.1 `sessions.subscribe`

协议说明：
- `docs/gateway/protocol.md:332`
- `ui/src/ui/controllers/sessions.ts:133`

含义：
- 为当前 WebSocket 客户端订阅会话索引变化事件。
- 一旦会话列表、会话元数据、最近活动等发生变化，Gateway 会通过 `sessions.changed` 通知客户端。

用户可感知功能：
- 侧边栏会话列表自动刷新
- 多端协同时，另一个端新开会话后当前端自动出现
- 会话名称、状态、最近活动时间实时更新
- 不需要用户手动刷新“会话列表”

适合开发的功能：
- 左侧“最近对话”列表
- 多设备同步会话列表
- 会话变更角标 / 实时更新提示

---

### 5.2 `sessions.list`

协议说明：
- `docs/gateway/protocol.md:332`
- `src/gateway/protocol/schema/sessions.ts:38`
- `ui/src/ui/controllers/sessions.ts:170`

含义：
- 返回当前 session 索引。
- 支持按活跃时间、数量、agent、label、关键字等方式过滤。
- 可以控制是否包含 global / unknown 等特殊会话。

重要参数含义：
- `limit`：最多返回多少条
- `activeMinutes`：只返回最近活跃的会话
- `includeDerivedTitles`：根据首条用户消息推导标题
- `includeLastMessage`：取最近一条消息摘要
- `label` / `search` / `agentId`：过滤条件

用户可感知功能：
- 显示“最近聊天记录”列表
- 在侧边栏显示每个会话标题和最后一条消息预览
- 按 agent、标签、搜索词筛选会话
- 只展示最近活跃的会话

适合开发的功能：
- ChatGPT/Claude 风格会话侧边栏
- 搜索历史会话
- 按机器人角色/工作区分类的会话列表
- “最近 7 天”/“今天”/“仅活跃会话” 过滤器

---

### 5.3 `sessions.messages.subscribe` / `sessions.messages.unsubscribe`

协议说明：
- `docs/gateway/protocol.md:335`
- `src/gateway/protocol/schema/sessions.ts:109`

含义：
- 针对某一个特定 session，订阅它的 transcript/message 事件。
- 比 `sessions.subscribe` 更细粒度，关注的是单个会话内容流，而不是整个会话索引变化。

用户可感知功能：
- 当前打开的对话窗口实时收到该会话的新消息
- 单会话级别的实时流更新
- 在大量会话存在时只监听当前打开那一个，提高前端效率

适合开发的功能：
- 多标签页聊天窗口
- 单个聊天页实时刷新
- 会话级实时消息订阅

备注：
- 当前 Dashboard 主要依赖 `chat` / `session.message` 事件和 `chat.history` 组合来保持 UI 一致，但如果你自己做客户端，单会话消息订阅也是值得关注的设计点。

---

### 5.4 `sessions.preview`

协议说明：
- `docs/gateway/protocol.md:337`
- `src/gateway/protocol/schema/sessions.ts:62`

含义：
- 对指定会话集合返回“受限预览”，而不是完整 transcript。
- 可以限制返回条数和字符数。

用户可感知功能：
- 鼠标悬停会话时显示对话摘要
- 会话卡片上展示 preview
- 搜索结果页里展示命中上下文片段

适合开发的功能：
- 会话 hover 预览
- 搜索结果 snippet
- “继续上次对话”卡片摘要

---

### 5.5 `sessions.resolve`

协议说明：
- `docs/gateway/protocol.md:339`
- `src/gateway/protocol/schema/sessions.ts:71`

含义：
- 把一个不完全明确的 session 标识解析为标准目标。
- 可以按 `key`、`sessionId`、`label`、`agentId` 等方式解析。

用户可感知功能：
- 用户输入一个会话别名或标签后，系统能自动定位正确会话
- 深链接打开会话时做规范化处理
- 会话跳转更稳健

适合开发的功能：
- URL 中用短 key / label 打开会话
- 全局搜索结果点击后跳转到会话
- 多种会话标识输入统一收敛为同一个 sessionKey

---

### 5.6 `sessions.create`

协议说明：
- `docs/gateway/protocol.md:340`
- `src/gateway/protocol/schema/sessions.ts:84`

含义：
- 显式创建一个新的 session。
- 可以指定：
  - `key`
  - `agentId`
  - `label`
  - `model`
  - `parentSessionKey`
  - `task`
  - `message`

用户可感知功能：
- 点击“新对话”时生成独立会话
- 从现有会话分叉出一个新线程
- 带预设任务的新工作流会话
- 为不同 agent 建立不同的专属对话

适合开发的功能：
- “New Chat”
- “Fork from here”
- “基于当前话题另开一个分支讨论”
- 带模板启动的新任务会话

---

### 5.7 `sessions.send`

协议说明：
- `docs/gateway/protocol.md:341`
- `src/gateway/protocol/schema/sessions.ts:97`

含义：
- 向一个已存在的 session 发送消息。
- 与 `chat.send` 相比，它更偏“会话目标显式已知”的发送方式。

用户可感知功能：
- 在指定会话中继续对话
- 后台任务会话追加消息
- 从会话列表中选择一个目标然后直接发消息

适合开发的功能：
- 多会话并行聊天
- 指定某个工作流会话继续执行
- 后台 agent 任务会话消息投递

备注：
- 当前 Dashboard 主聊天更直接使用 `chat.send`。
- 如果你的应用对“会话是第一公民”建模更强，可以更多考虑 `sessions.send`。

---

### 5.8 `sessions.steer`

协议说明：
- `docs/gateway/protocol.md:342`

含义：
- 对正在运行的 session 做“中断并改道”的控制。
- 可以理解为不中断整个上下文，而是对当前进行中的执行插入新的指导。

用户可感知功能：
- 用户在生成中途说“换个方向”
- “别继续这个回答了，改成表格输出”
- “停止当前思路，按另一个要求继续”

适合开发的功能：
- 生成中实时转向
- AI 输出纠偏
- 中途编辑任务目标

这是高级交互功能，适合做“比普通聊天更强”的控制式 Chatbot。

---

### 5.9 `sessions.abort`

协议说明：
- `docs/gateway/protocol.md:343`
- `src/gateway/protocol/schema/sessions.ts:123`

含义：
- 中止指定 session 的当前活动运行。
- 可选带 `runId` 指定要终止哪一个 run。

用户可感知功能：
- “停止生成”按钮
- 中止长时间运行的任务
- 用户主动取消当前回答

适合开发的功能：
- Stop 按钮
- 取消后台长任务
- 中断错误方向的运行

---

### 5.10 `sessions.patch`

协议说明：
- `docs/gateway/protocol.md:344`
- `src/gateway/protocol/schema/sessions.ts:131`
- `ui/src/ui/controllers/sessions.ts:208`

含义：
- 更新某个 session 的元数据或运行偏好。

从 schema 看，可更新内容包括：
- `label`
- `thinkingLevel`
- `fastMode`
- `verboseLevel`
- `reasoningLevel`
- `responseUsage`
- `model`
- `spawnedBy`
- 一些 subagent / group / exec 相关设置

用户可感知功能：
- 修改会话标题
- 为某个会话切换模型
- 给某个会话设置更高/更低 thinking level
- 打开或关闭 fast mode
- 让某个会话显示更详细日志 / 推理风格

适合开发的功能：
- 会话重命名
- “此对话使用 GPT-5.4 / Sonnet 4.6”
- 每会话独立的模型偏好
- 高级用户的 per-session 设置面板

这是非常适合做差异化体验的一组接口。

---

### 5.11 `sessions.reset`

协议说明：
- `docs/gateway/protocol.md:345`
- `src/gateway/protocol/schema/sessions.ts:175`

含义：
- 对指定 session 做重置。
- 可以带 `reason: "new" | "reset"`。

用户可感知功能：
- 清空当前对话上下文
- 从当前聊天框开始一个“新对话”
- 保留壳子但不继承前文

适合开发的功能：
- “New Chat”
- “重置当前上下文”
- 快速开始新一轮问答

在实际产品上，这通常比真正删除 session 更常用。

---

### 5.12 `sessions.delete`

协议说明：
- `docs/gateway/protocol.md:345`
- `src/gateway/protocol/schema/sessions.ts:183`
- `ui/src/ui/controllers/sessions.ts:264`

含义：
- 删除一个 session 记录。
- 可选 `deleteTranscript` 控制是否删除 transcript。

用户可感知功能：
- 删除聊天记录
- 清理无用任务会话
- 批量删除旧会话

适合开发的功能：
- 侧边栏删除会话
- 多选删除历史
- 自动清理陈旧会话

---

### 5.13 `sessions.compact`

协议说明：
- `docs/gateway/protocol.md:345`
- `src/gateway/protocol/schema/sessions.ts:193`

含义：
- 对 session 做压缩，减少上下文长度。
- 目的是在不完全丢失历史的前提下缩短上下文，提高后续运行效率。

用户可感知功能：
- 长对话保持“还能继续聊”
- 很长的任务线程自动变轻
- 节省 token / 提高后续响应稳定性

适合开发的功能：
- “压缩上下文”按钮
- 超长对话自动整理
- 长任务会话的上下文维护

---

### 5.14 `sessions.compaction.list`

协议说明：
- `src/gateway/protocol/schema/sessions.ts:201`
- `ui/src/ui/controllers/sessions.ts:67`

含义：
- 列出某个 session 的 compaction checkpoint。
- 这些 checkpoint 描述了压缩前后引用关系、时间、原因等。

用户可感知功能：
- 查看“这个会话被压缩过几次”
- 查看上下文压缩历史
- 追踪长对话的演进过程

适合开发的功能：
- 会话压缩历史列表
- 会话维护面板

---

### 5.15 `sessions.compaction.get`

协议说明：
- `src/gateway/protocol/schema/sessions.ts:208`

含义：
- 获取某个 checkpoint 的详细信息。

用户可感知功能：
- 查看一次上下文压缩的详情
- 了解压缩前后保留了什么

适合开发的功能：
- checkpoint 详情页
- 调试长会话压缩行为

---

### 5.16 `sessions.compaction.branch`

协议说明：
- `src/gateway/protocol/schema/sessions.ts:216`
- `ui/src/ui/controllers/sessions.ts:296`

含义：
- 从某个 compaction checkpoint 分叉出一个新的子 session。

用户可感知功能：
- 从历史某个阶段重新开一个分支继续聊
- 从压缩前状态衍生出另一个探索方向

适合开发的功能：
- “从这里分叉会话”
- 历史 checkpoint 的平行探索
- AI 任务的多分支实验

这是非常适合高阶用户或研究型产品的功能。

---

### 5.17 `sessions.compaction.restore`

协议说明：
- `src/gateway/protocol/schema/sessions.ts:224`
- `ui/src/ui/controllers/sessions.ts:102`

含义：
- 将一个 session 恢复到某个 checkpoint 状态。

用户可感知功能：
- 撤销一次上下文压缩
- 回到某个历史版本继续对话
- 从错误整理结果中回滚

适合开发的功能：
- “恢复到这个版本”
- 会话版本回滚
- 历史上下文恢复

---

### 5.18 `sessions.usage`

协议说明：
- `docs/gateway/protocol.md:247`
- `src/gateway/protocol/schema/sessions.ts:285`
- `ui/src/ui/controllers/usage.ts:185`

含义：
- 统计 session 的 usage 数据。
- 支持时间范围、时区解释、限制数量、上下文权重信息。

用户可感知功能：
- 查看每个会话用了多少 token / 成本
- 统计近期使用量
- 找出最“贵”的对话

适合开发的功能：
- Usage 面板
- 成本审计
- 按时间统计聊天开销

---

### 5.19 `sessions.usage.timeseries`

协议说明：
- `docs/gateway/protocol.md:248`
- `ui/src/ui/controllers/usage.ts:263`

含义：
- 返回某个 session 的 usage 时间序列。

用户可感知功能：
- 看这个会话的消耗曲线
- 识别哪一段对话最耗费 token

适合开发的功能：
- usage 折线图
- 对话成本趋势图

---

### 5.20 `sessions.usage.logs`

协议说明：
- `docs/gateway/protocol.md:249`
- `ui/src/ui/controllers/usage.ts:271`

含义：
- 返回某个 session 的 usage 日志明细。

用户可感知功能：
- 查看每一步调用的明细开销
- 审计一段对话里的成本构成

适合开发的功能：
- usage 明细表
- 成本审计日志
- 排障分析视图

---

## 6. 如果你只做一个“用户能感知”的 Chatbot，最值得先做的会话功能

如果你想优先做用户最容易感知价值的功能，而不是一下子实现所有 session 接口，我建议按下面顺序：

### 第一层：基础可用
- `sessions.list`
- `chat.history`
- `chat.send`
- `sessions.reset`
- `sessions.delete`

能做出的用户功能：
- 会话列表
- 聊天历史
- 新建/重置聊天
- 删除聊天记录

### 第二层：体验增强
- `sessions.subscribe`
- `sessions.patch`
- `sessions.abort`

能做出的用户功能：
- 自动刷新的会话列表
- 会话重命名
- per-session 模型/模式设置
- 停止生成

### 第三层：高级能力
- `sessions.preview`
- `sessions.resolve`
- `sessions.create`
- `sessions.compact`
- `sessions.compaction.*`
- `sessions.usage*`

能做出的用户功能：
- 会话摘要 / 搜索 / 深链接
- 从历史点分叉对话
- 长对话上下文压缩与恢复
- 成本与 usage 可视化

---

## 7. 推荐的接口组合

## 方案 A：最像 Dashboard 的 Chatbot

使用：
- WS `connect`
- `sessions.subscribe`
- `sessions.list`
- `chat.history`
- `chat.send`
- `sessions.reset`
- 监听 `chat` / `session.message` / `sessions.changed`

适合：
- 原生 Web 客户端
- 管理台式聊天界面
- 多会话体验

## 方案 B：更强的生产级聊天客户端

在方案 A 的基础上，再加：
- `sessions.patch`
- `sessions.abort`
- `agent.identity.get`
- `models.list`
- `sessions.preview`
- `sessions.usage*`

适合：
- 需要丰富设置和数据分析的产品
- 面向高级用户的桌面/Web Chatbot

## 方案 C：只想快速接入聊天

直接使用：
- `/v1/chat/completions`
- `/v1/responses`

适合：
- 原型产品
- 接 OpenAI SDK 的服务层
- 不强依赖 session/event 的场景

---

## 8. 最后建议

如果你的目标是“做一个尽量复用 OpenClaw 能力的定制 Chatbot”，推荐顺序是：

1. 先实现 WebSocket `connect`
2. 先打通 `chat.send` + `chat.history`
3. 再补 `sessions.list` + `sessions.subscribe`
4. 然后补 `sessions.reset` + `sessions.patch`
5. 最后再引入 `sessions.compaction.*` 与 `sessions.usage*`

这样你能较快做出一个：

- 看起来像 Dashboard
- 具备多会话能力
- 支持实时更新
- 用户能明显感知差异化体验

的 Chatbot。
