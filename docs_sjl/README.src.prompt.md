# Hermes Agent Prompt 机制源码导读

这份文档说明 Hermes Agent 中 system prompt 的机制。

结论：

> system prompt 内容按职责拆在多个模块里，最后由 agent 初始化和运行流程组合、缓存、发送给模型。

---

## 1. 通用 system prompt 内容在哪里

主要文件：

- `agent/system_prompt.py`

这里放的是通用身份、长期行为规则、工具使用规则、memory/skill/context 等系统级提示。

典型内容包括：

- `DEFAULT_AGENT_IDENTITY`
- `HERMES_AGENT_HELP_GUIDANCE`
- `MEMORY_GUIDANCE`
- `SESSION_SEARCH_GUIDANCE`
- `SKILLS_GUIDANCE`
- `TOOL_USE_ENFORCEMENT_GUIDANCE`
- `TASK_COMPLETION_GUIDANCE`
- `PARALLEL_TOOL_CALL_GUIDANCE`

---

## 2. 项目上下文文件如何进入 prompt

主要文件：

- `agent/system_prompt.py`

关键函数：

- `build_context_files_prompt()`
- `load_soul_md()`
- `_load_hermes_md()`
- `_load_agents_md()`
- `_load_claude_md()`
- `_load_cursorrules()`

它们负责发现并读取这些上下文文件：

- `SOUL.md`
- `.hermes.md`
- `HERMES.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.cursorrules`
- `.cursor/rules/*.mdc`

这些文件会被作为 project context 注入 system prompt。

Hermes Agent 在这里还做了两类额外处理：

- 截断过长内容，避免上下文文件无限膨胀
- 扫描 prompt injection / promptware 风险，命中后不会把原文塞进 system prompt

---

## 3. coding agent 的提示在哪里

主要文件：

- `agent/coding_context.py`

这里处理 Hermes Agent 的 coding posture，也就是当 Hermes 判断当前会话处在代码工作区时，给模型注入更像“编码 agent”的行为约束。

典型内容包括：

- `CODING_AGENT_GUIDANCE`
- `build_coding_workspace_block()`
- `coding_system_blocks()`
- `RuntimeMode.system_blocks()`

`CODING_AGENT_GUIDANCE` 里定义的是编码任务的工作方式，例如：

- 先读取相关文件再修改
- 使用工具编辑文件，不用聊天内容替代文件修改
- 跟随项目已有风格
- 运行测试或构建验证
- 不主动提交、推送、改写历史

`build_coding_workspace_block()` 会生成工作区快照，例如：

- workspace root
- git branch
- git status
- recent commits
- manifest / package manager / verify commands

这部分对应当前项目里如果未来要拆分“通用 agent 能力”和“业务/编码姿态”的位置。

---

## 4. 最终在哪里组装进 agent

主要文件：

- `agent/agent_init.py`

`agent_init.py` 中的 `init_agent(...)` 是 agent 初始化入口。它负责把模型、provider、toolsets、memory、context、runtime mode、ephemeral system prompt 等状态准备好。

它不是只维护一个简单的 `sections` 列表，而是把多个模块的结果组合为 agent 会话状态，并配合缓存机制使用。

相关职责包括：

- 初始化 `agent.ephemeral_system_prompt`
- 初始化 `agent.enabled_toolsets` / `agent.disabled_toolsets`
- 初始化 context compressor
- 初始化 prompt caching 相关状态
- 解析 coding runtime mode
- 准备最终 system prompt 构建所需的上下文

---

## 5. 为什么不把所有内容集中写在一个方法里

Hermes Agent 非常重视 prompt cache。

因此它避免在长会话中频繁重建 system prompt，也避免把可变内容随意塞进 system prompt。

关键设计原则是：

```text
稳定、长期有效的基础规则
  -> system prompt

用户显式触发的 skill
  -> user message 注入

模型按需使用的 skill
  -> skills_list / skill_view 工具渐进读取

项目上下文文件
  -> session start 时读取并注入

临时请求、任务参数、一次性上下文
  -> user message 或 tool result
```

这也是为什么 Hermes Agent 中的 prompt 相关代码分布在多个模块，而不是集中在一个 `_system_prompt()` 方法里。

---

## 6. 一句话总结

Hermes Agent 的 system prompt 分布在多个模块里，每个模块负责不同的职责：

- `agent/system_prompt.py`
  - 通用身份、通用规则、memory/skill/context 提示
- `agent/coding_context.py`
  - coding posture 和 workspace snapshot
- `agent/agent_init.py`
  - agent 初始化、toolsets、缓存、runtime mode 和最终 prompt 状态准备
- `agent/skill_commands.py`
  - slash command 显式加载 skill，作为 user message 注入，而不是修改 system prompt
- `tools/skills_tool.py`
  - `skills_list` / `skill_view`，支持模型按需发现和读取 skill
