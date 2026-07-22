# Hermes Agent 源码导读

这份文档是面向源码阅读的“地图”，重点不在功能宣传，而在于回答三个问题：

1. 这个工程的核心运行链路是什么
2. 每个关键模块负责什么
3. 从哪里开始读，效率最高

先给一个整体判断：

> 这个仓库不是单一 CLI 工具，而是一个以 `AIAgent` 为内核、挂了多种前端和扩展系统的 agent 平台。

它的主线可以概括为：

```text
前端入口（CLI / Gateway / TUI / Desktop）
  -> AIAgent
  -> model_tools
  -> tools registry / plugins
  -> 会话与状态持久化
```

---

## 1. 核心心脏在哪里

最核心的入口仍然是 `run_agent.py` 里的 `AIAgent`：

- `run_agent.py`
- `agent/agent_init.py`
- `agent/conversation_loop.py`

不过现在的 `AIAgent` 已经被明显“瘦身”：

- `AIAgent.__init__` 实际转发到 `agent/agent_init.py:init_agent`
- `AIAgent.run_conversation()` 实际转发到 `agent/conversation_loop.py:run_conversation`

这说明项目在做一件持续演进的事：

- 对外保留稳定入口
- 对内把 `run_agent.py` 里超长逻辑逐步拆到 `agent/` 目录

所以读这个仓库时，不要只盯着 `run_agent.py` 的文件体量，要顺着 forwarder 看真实实现。

---

## 2. 一次用户请求是怎么跑起来的

推荐用下面这条链理解：

```text
用户输入
  -> 前端入口接收消息
  -> 创建/复用 AIAgent
  -> run_conversation 主循环
  -> 调模型
  -> 解析 tool calls
  -> model_tools.handle_function_call
  -> tools.registry.dispatch
  -> 工具执行结果回写消息流
  -> SessionDB 持久化
  -> 返回最终响应
```

### 2.1 前端层收消息

几个主要入口：

- `cli.py`：传统交互式 CLI
- `gateway/run.py`：Telegram / Discord / Slack / WhatsApp / Signal 等消息平台入口
- `tui_gateway/server.py`：TUI 的 Python 后端
- `apps/desktop/`：桌面端独立聊天界面

它们是不同的“壳”，但共用同一套 agent 内核。

### 2.2 进入 AIAgent

`AIAgent` 负责持有一次会话运行所需的核心状态，例如：

- 模型与 provider
- API mode
- enabled/disabled toolsets
- session_id
- memory / context 相关状态
- callbacks
- iteration budget
- checkpoint 配置

这一层更像一个“运行态容器”，而不是简单的业务类。

### 2.3 对话主循环

真正的 agent 回合逻辑在：

- `agent/conversation_loop.py`

这里负责：

- 构造或恢复 system prompt
- 加载 memory context
- 组织 messages
- 调用模型接口
- 处理 tool calls
- 失败重试与 provider fallback
- 上下文压缩
- 会话持久化
- 插件 hook 触发

这是整个工程最值得精读的文件之一。

---

## 3. run_agent.py 的真实角色

虽然 `run_agent.py` 很大，但它现在更像：

- 稳定的公共入口
- 兼容测试 patch surface
- 一些跨模块共用的辅助逻辑容器

典型特征：

- 保留 `AIAgent` 类定义
- 保留 `main()` 直跑入口
- 很多复杂逻辑转发给 `agent/*`
- 对测试暴露若干可 patch 的符号

也就是说，这个文件既是“门面”，也是历史兼容层。

---

## 4. 工具系统：这是 Hermes 的第二核心

Hermes 不是单靠模型输出工作的，它是一个强工具调用型 agent。这里有三层要分清。

### 4.1 tool 注册中心：`tools/registry.py`

这是最底层的工具基础设施。

模式是：

1. 每个 `tools/*.py` 在 import 时调用 `registry.register(...)`
2. `model_tools.py` 启动时执行 `discover_builtin_tools()`
3. registry 收集所有工具的：
   - name
   - schema
   - handler
   - toolset
   - availability check
   - 描述与元数据

这是一种“自注册”设计，避免集中式维护长列表。

### 4.2 调度层：`model_tools.py`

`model_tools.py` 是工具编排层，不是工具实现层。

它负责：

- 启动 builtin tool discovery
- 封装 `get_tool_definitions()`
- 根据 toolset 解析当前会话可见工具
- 统一处理 `handle_function_call()`
- 插件 hook 前后处理
- 某些 bridge tool 的分流，例如 tool search / tool describe / tool call

这层可以理解成：

> “LLM 看见的工具世界” 和 “真实 registry 里的所有工具” 之间的那层转换器

### 4.3 工具暴露控制：`toolsets.py`

工具注册了，不代表当前 agent 一定能用。

`toolsets.py` 定义了：

- 默认核心工具 `_HERMES_CORE_TOOLS`
- 基础 toolset：`web`、`browser`、`file`、`memory`、`terminal` 等
- 组合型 toolset
- 某些场景化工具可见性控制

所以工具的生效流程是：

```text
工具实现存在
  -> registry 已注册
  -> 被某个 toolset 引用
  -> 当前 session 启用了该 toolset
  -> 最终进入模型可见 schema
```

这点很关键。很多“为什么模型看不到某工具”的问题，都要从这里查。

---

## 5. 状态层：SessionDB 不是附属件，而是主能力

状态与会话持久化由：

- `hermes_state.py`

中的 `SessionDB` 承担。

它不只是“聊天记录存档”，而是多个能力的底座：

- session 元信息
- message 存储
- `/resume`
- `session_search`
- FTS5 全文检索
- system prompt 快照恢复
- 多进程共享状态

实现方案：

- SQLite 持久化
- WAL 模式        # 解决的是“并发读写和持久运行”
- FTS5 全文检索    # 解决的是“历史消息的高性能全文搜索”
- 每个方法独立 cursor
- 写入冲突靠应用层 jitter retry，而不是只靠 SQLite busy timeout

### 5.1 设计特征

这部分代码明显是按“长期运行服务”来打磨的。

**并发与持久化**这块，Hermes 采用 SQLite 的 **WAL 模式**（Write-Ahead Logging，预写日志）：

- 写入先追加到 `*.wal` 文件，再由 checkpoint 合并回主库
- 读写并发优于传统 `DELETE` journal，适合 CLI、gateway、TUI 共享同一个 `state.db`
- 每隔一段写入会主动做 `wal_checkpoint(TRUNCATE)`，避免 WAL 文件无限增长
- 对 NFS / SMB / 某些 FUSE 这类不兼容 WAL 的文件系统，自动回退到 `journal_mode=DELETE`

同时它没有完全依赖 SQLite 默认的 busy timeout，而是自己做了写锁竞争处理：

- application-level jitter retry
- `BEGIN IMMEDIATE`
- 遇到 `database is locked` 时随机退避再重试，降低多进程争锁时的 convoy effect

**全文检索**这块，Hermes 用的是 SQLite 的 **FTS5**：

- FTS5 可用性探测
- 维护 `messages_fts` 虚拟表作为全文索引
- 通过 trigger 把 `messages` 表的变更同步到 FTS 表
- 索引内容不只包含 `content`，还包含 `tool_name` 和 `tool_calls`
- 额外维护 `messages_fts_trigram`，专门改善 CJK / 子串搜索

如果当前 Python/SQLite 运行时不支持 FTS5，Hermes 会关闭全文检索，但保留核心会话存储能力，避免“搜索坏了导致整个状态层不可用”。

说明作者已经把 CLI、gateway、多进程/多线程并发这类问题真正跑过一轮，不是只写了 happy path。

### 5.2 为什么它重要

Hermes 的很多高级能力都靠它：

- 历史恢复
- 跨 session 检索
- prompt cache 复用
- gateway 与 CLI 共享状态

所以它更像 Hermes 的**会话与检索基础设施**。如果只把它当“日志数据库”看，会低估这个模块的重要性。

---

## 6. CLI / Gateway / TUI / Desktop 的关系

这个项目容易让人误会成“很多套重复实现”，但实际上核心设计是：

- 多前端
- 单内核

### 6.1 CLI：`cli.py`

`HermesCLI` 是终端 REPL 层，负责：

- 展示
- 输入循环
- slash command 交互
- 调用 `AIAgent`
- session 切换与恢复

它的职责主要是“交互壳”，不是 agent 推理内核。

### 6.2 Gateway：`gateway/run.py`

这是消息平台网关。

它负责：

- 启动多平台 adapter
- 生命周期管理
- 接收平台消息事件
- 安全过滤/降噪
- 调用 agent
- 返回平台侧消息

这一层已经是“长生命周期 daemon”形态，所以能看到不少防崩设计：

- loop exception handler
- transient network error 吞吐保护
- 用户可见错误脱敏

### 6.3 TUI：`ui-tui/` + `tui_gateway/server.py`

TUI 的架构是：

- 前端：Node + Ink
- 后端：Python `tui_gateway`
- 通信：stdio JSON-RPC

这意味着：

- TUI 不是对 CLI 的简单皮肤替换
- 也不是把 agent 逻辑搬到 TypeScript
- 它是一个独立前端，通过 RPC 驱动 Python agent 内核

### 6.4 Desktop

桌面端是独立聊天界面，不直接嵌入 `hermes --tui`，而是通过 gateway/RPC 风格后端交互。

所以整个项目前端层次大概是：

```text
CLI                  -> 直接调用 Python agent
Messaging Gateway    -> 平台消息适配后调用 Python agent
TUI                  -> TS 前端 + Python JSON-RPC backend
Desktop              -> 独立 GUI chat surface + 后端网关
```

---

## 7. slash command 系统做得比较统一

命令元数据的单一事实源在：

- `hermes_cli/commands.py`

核心对象是：

- `COMMAND_REGISTRY`
- `CommandDef`

这层统一派生出：

- CLI dispatch
- gateway command help
- Telegram command 菜单
- Slack 子命令映射
- autocomplete

这个设计的优点很明显：

- 新命令元信息只维护一处
- alias 维护简单
- 多前端对命令的认知一致

这一层在工程结构上是比较“干净”的。

---

## 8. 插件系统：Hermes 的主要扩展面

### 8.1 通用插件：`hermes_cli/plugins.py`

这是通用插件框架，负责发现和加载：

- bundled plugins
- user plugins
- project plugins
- pip entry-point plugins

插件可以：

- 注册 hook
- 注册工具
- 注册 CLI 子命令

常见 hook 包括：

- `pre_tool_call`
- `post_tool_call`
- `pre_llm_call`
- `post_llm_call`
- `on_session_start`
- `on_session_end`
- `pre_gateway_dispatch`

所以它不是“配置型插件”，而是真正能参与 agent 生命周期的扩展面。

### 8.2 provider 插件：`providers/__init__.py`

模型 provider 的发现系统是单独的一套：

- `plugins/model-providers/<name>/`
- `$HERMES_HOME/plugins/model-providers/<name>/`
- 兼容旧版 `providers/*.py`

它的特点：

- lazy discovery
- last-writer-wins 覆盖机制
- 用户插件可覆盖内置 provider

也就是说：

- 通用插件系统不等于 provider 系统
- provider 只是“插件化”的另一条专门通道

### 8.3 其他插件族

仓库里还能看到一些专门目录：

- `plugins/memory/`
- `plugins/context_engine/`
- `plugins/image_gen/`
- `plugins/platforms/`

这说明 Hermes 的插件化不是单点设计，而是多条能力链都在逐步插件化。

---

## 9. provider 与模型路由层的理解方式

这个项目的模型接入不是“写死 OpenAI 风格接口”那么简单。

它要处理：

- 多 provider
- 多 base_url
- 不同 api_mode
- 动态 fallback
- custom provider profile

因此源码里你会看到几层分工：

- `run_agent.py` / `agent_init.py`：运行态选择和组装
- `providers/`：provider profile 注册与发现
- `plugins/model-providers/`：provider 实现目录
- `agent/*` 中若干辅助模块：metadata、pricing、timeouts、retry 等

这一层的复杂度不低，是整个系统的另一个“平台级子系统”。

---

## 10. 这个仓库的工程风格

读下来大致有几个特征。

### 10.1 历史包袱明显

你会看到很多：

- 大文件
- 兼容旧接口的 forwarder
- 为测试 patch surface 保留符号
- “现在的实现已抽走，但入口还保留在原文件”

这意味着它不是一个从零设计得非常纯粹的新仓库，而是一个不断增长、不断重构中的真实项目。

### 10.2 但演进方向是清楚的

架构上能看出几个明确趋势：

- 从大文件向 `agent/` 内部模块拆分
- 从手工静态 wiring 向 registry / plugin discovery 迁移
- 从单 CLI 走向多前端共享内核
- 从“功能堆叠”向“扩展面清晰化”推进

### 10.3 偏实战，重稳定性

大量代码是在处理真实运行环境问题，而不是只写核心 happy path，例如：

- 持久 event loop
- gateway transient error 兜底
- SQLite contention retry
- crash panic hook
- tool availability cache
- 动态 schema override

所以这不是一个“很轻、很整洁”的教学型工程，而是一个功能密、入口多、长期运行场景驱动的工程。

---

## 11. 推荐阅读顺序

如果你的目标是“最快读懂”，建议按下面顺序：

1. `README.md`
2. `run_agent.py`
3. `agent/agent_init.py`
4. `agent/conversation_loop.py`
5. `model_tools.py`
6. `tools/registry.py`
7. `toolsets.py`
8. `hermes_state.py`
9. `cli.py`
10. `gateway/run.py`
11. `hermes_cli/commands.py`
12. `hermes_cli/plugins.py`
13. `providers/__init__.py`

### 为什么这样排

- 先抓主链路：agent 如何跑
- 再抓工具系统：模型如何调用能力
- 再抓持久化：状态如何保存
- 最后看前端与扩展：不同入口如何复用内核

这样不会一开始就淹没在平台细节里。

---

## 12. 读源码时最该抓住的四层

如果只记住四个关键词，建议是：

```text
AIAgent 生命周期
  -> 对话循环
  -> 工具注册/分发
  -> 会话/插件/Provider 扩展
```

它们分别对应：

- `run_agent.py` + `agent/agent_init.py`
- `agent/conversation_loop.py`
- `model_tools.py` + `tools/registry.py` + `toolsets.py`
- `hermes_state.py` + `hermes_cli/plugins.py` + `providers/__init__.py`

这四层抓住了，整个项目的骨架就基本清楚了。

---

## 13. 一句话总结

Hermes Agent 本质上是在做一个：

> 可插拔、多前端、带工具调用、带持久记忆、可跨平台运行的 agent runtime

所以阅读时不要把它当成：

- 一个普通 CLI 程序
- 一个单纯的聊天机器人
- 一组散落工具脚本

更准确的理解是：

- 它是一个 agent 平台
- `AIAgent` 是内核
- tool / plugin / provider / session 是主要支撑系统

---

## 14. 后续可继续深挖的方向

如果后面继续扩展这份文档，建议优先拆以下专题：

1. `run_conversation` 主循环逐段解读
2. `tools/` 工具体系设计与注册样式
3. `gateway/` 多平台消息接入模型
4. `plugins/` 插件开发入口与生命周期
5. `SessionDB` 的表结构与检索流程
6. TUI 的 JSON-RPC 通信模型

这些方向都值得单独成篇。
