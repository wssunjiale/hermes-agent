# Hermes Agent Skill 机制源码导读

这份文档专门解释 Hermes 是如何判断“什么时候需要 skill、需要哪个 skill”的。

先给结论：

> Hermes 核心里没有一个确定性的“自然语言 -> skill”分类器。  
> skill 的使用主要分两类：用户或配置显式指定；模型通过 `skills_list` / `skill_view` 工具自行发现和选择。

也就是说，Hermes 不会在核心代码里写死“看到 GitHub 就加载 github skill”这类规则。

---

## 1. Skill 的两种触发方式

Hermes 里 skill 的触发主要有两条路径：

```text
显式触发
  -> slash command / 启动参数 / gateway channel 绑定
  -> 直接加载指定 skill

模型自主发现
  -> 模型看到 skills_list / skill_view 工具
  -> 调 skills_list 看可用 skill
  -> 根据 name / description / category 判断
  -> 调 skill_view 读取完整 SKILL.md
```

这两条路径解决的问题不同：

- 显式触发：用户或系统已经知道要用哪个 skill
- 自主发现：模型在任务过程中判断是否需要某个 skill

---

## 2. Skill 工具默认暴露给模型

skill 能被模型主动发现，是因为下面三个工具在 core tools 里：

- `skills_list`
- `skill_view`
- `skill_manage`

位置在：

- `toolsets.py`

关键代码：

```python
_HERMES_CORE_TOOLS = [
    ...
    "skills_list", "skill_view", "skill_manage",
    ...
]
```

这意味着普通 agent session 默认能看到 skill 工具。

`toolsets.py` 里还单独定义了 `skills` toolset：

```python
"skills": {
    "description": "Access, create, edit, and manage skill documents with specialized instructions and knowledge",
    "tools": ["skills_list", "skill_view", "skill_manage"],
    "includes": []
}
```

所以模型自主选择 skill 的前提是：

```text
skills_list / skill_view 在当前 session 的 tool schema 里
  -> 模型可以调用它们
  -> 模型根据返回内容判断下一步
```

---

## 3. 模型如何知道有哪些 skill

模型不会一开始拿到所有 `SKILL.md` 的完整内容。

它先通过：

- `tools/skills_tool.py:skills_list`

获取轻量列表。

`skills_list()` 返回的是最小元数据：

- name
- description
- category
- count
- categories

代码注释里也明确写了这是 progressive disclosure：

```python
def skills_list(category: str = None, task_id: str = None) -> str:
    """
    List all available skills (progressive disclosure tier 1 - minimal metadata).

    Returns only name + description to minimize token usage. Use skill_view() to
    load full content, tags, related files, etc.
    """
```

设计意图是：

```text
先给模型一个小目录
  -> 模型根据描述选中候选 skill
  -> 再读取完整内容
```

这样避免每轮把所有 skill 全量塞进上下文，保护 prompt cache 和 token 成本。

---

## 4. 模型如何加载某个 skill

模型选定一个 skill 后，会调用：

- `tools/skills_tool.py:skill_view`

`skill_view()` 会加载完整 skill 内容，支持几种名字解析方式：

- 直接路径，例如 `mlops/axolotl`
- skill 目录名
- frontmatter 里的 `name`
- legacy flat `<name>.md`
- plugin skill 的 `plugin:skill` 形式

核心行为是：

```text
skill_view(name)
  -> 查找匹配的 SKILL.md
  -> 读取 frontmatter 和正文
  -> 做平台/禁用/安全检查
  -> 返回 content、description、tags、related_skills、linked_files 等
```

如果有 supporting files，例如：

- `references/`
- `templates/`
- `scripts/`
- `assets/`

`skill_view()` 会在返回里给出 `linked_files`，模型可以再次调用：

```text
skill_view(name, file_path="references/api.md")
```

来读取具体文件。

这也是分层加载：

```text
skills_list
  -> skill_view(name)
  -> skill_view(name, file_path)
```

---

## 5. Slash command 是如何变成 skill 的

显式触发的核心文件是：

- `agent/skill_commands.py`

### 5.1 扫描 skill 生成 slash command

入口：

```python
def scan_skill_commands() -> Dict[str, Dict[str, Any]]:
```

它会扫描：

- `~/.hermes/skills/`
- `skills.external_dirs` 配置的外部目录

并查找每个 skill 的：

- `SKILL.md`

然后读取 frontmatter：

- `name`
- `description`

再把 name 规范化成 slash command：

```text
My Skill_Name
  -> /my-skill-name
```

代码里会做这些处理：

- 小写
- 空格和下划线转成 `-`
- 移除不合法字符
- 合并多个连续 `-`

最后生成类似：

```python
{
    "/github-pr-workflow": {
        "name": "github-pr-workflow",
        "description": "...",
        "skill_md_path": ".../SKILL.md",
        "skill_dir": "...",
    }
}
```

### 5.2 CLI 里如何触发

位置：

- `cli.py`

关键逻辑：

```python
elif base_cmd in skill_commands:
    user_instruction = cmd_original[len(base_cmd):].strip()
    msg = build_skill_invocation_message(
        base_cmd, user_instruction, task_id=self.session_id
    )
```

也就是说，用户输入：

```text
/github-pr-workflow review this PR
```

会变成：

```text
加载 github-pr-workflow skill
并把 "review this PR" 作为用户附加指令
```

### 5.3 TUI / Desktop 里如何触发

TUI / Desktop 通过 `tui_gateway/server.py` 的 RPC 路径处理。

关键路径是：

- `commands.catalog`
- `complete.slash`
- `command.dispatch`

`command.dispatch` 里会：

```python
cmds = scan_skill_commands()
key = f"/{name}"
if key in cmds:
    msg = build_skill_invocation_message(...)
    return {
        "type": "skill",
        "message": msg,
        "name": ...
    }
```

Desktop 前端拿到 `{type: "skill", message}` 后，会把这个 message 当作普通 prompt 提交。

---

## 6. Skill 被加载后实际塞到哪里

slash command 加载 skill 时，不是修改 system prompt。

核心函数：

- `agent/skill_commands.py:build_skill_invocation_message`

它会生成一段用户消息：

```text
[IMPORTANT: The user has invoked the "xxx" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]

<SKILL.md content>

[Skill directory: ...]

The user has provided the following instruction alongside the skill invocation: ...
```

这个设计点很重要：

> slash skill 注入为 user message，而不是中途修改 system prompt。

原因是 Hermes 非常重视 prompt caching：

- 长会话复用 cached prefix
- 中途改 system prompt 会破坏缓存
- 所以 slash skill 用普通消息注入

这也解释了为什么 `reload_skills()` 只刷新 slash command cache，不重建 system prompt。

---

## 7. 启动时预加载 skill

另一种显式方式是启动时指定 skill。

CLI 入口：

- `cli.py:_parse_skills_argument`
- `cli.py` 里调用 `build_preloaded_skills_prompt`

TUI 入口：

- `tui_gateway/server.py:_parse_tui_skills_env`
- `_make_agent()` 里调用 `build_preloaded_skills_prompt`

核心函数：

```python
def build_preloaded_skills_prompt(
    skill_identifiers: list[str],
    task_id: str | None = None,
) -> tuple[str, list[str], list[str]]:
```

这条路径和 slash command 不同：

```text
启动时明确指定 skill
  -> 构造 skills_prompt
  -> 拼到 session system_prompt
```

适用场景是：

- worker 启动时固定带某个能力
- TUI/CLI session 从一开始就采用某个工作流
- Kanban / 自动化任务希望 session 全程遵守某个 skill

如果多个 skill 里有一部分找不到：

- 至少加载成功一个：跳过缺失项并 warning
- 全部缺失：抛错

---

## 8. Gateway 的自动 skill 绑定

Messaging gateway 还有一条自动加载路径。

位置：

- `gateway/run.py`

逻辑大意：

```text
event.auto_skill 存在
且这是新 session
  -> 加载 auto_skill
  -> 把 skill 内容拼到 event.text 前面
  -> 再进入 agent
```

代码注释说明了用途：

```text
Auto-load skill(s) for topic/channel bindings
```

也就是：

- Telegram topic
- Discord channel_skill_bindings
- 其他 channel/topic 绑定

可以让某个渠道默认使用某个 skill。

注意它只在新 session 注入：

```text
ongoing conversations already have the skill content in their conversation history
```

这避免每条消息重复塞同一个 skill。

---

## 9. 哪些 skill 会被过滤掉

`scan_skill_commands()` 和 `skills_list()` 不是无条件暴露所有 skill。

常见过滤条件有三类。

### 9.1 平台不匹配

位置：

- `tools/skills_tool.py:skill_matches_platform`
- `agent/skill_commands.py` 调用它

如果 skill frontmatter 声明只支持某些平台，而当前平台不匹配，会跳过。

### 9.2 runtime 环境不匹配

位置：

- `tools/skills_tool.py:skill_matches_environment`

用于过滤只适合特定运行环境的 skill，例如：

- kanban
- docker
- s6

注释里说明这是 offer-time filtering：

```text
Offer-time only; explicit load bypasses.
```

也就是说，普通列表/补全里不展示不相关 skill，但显式加载路径仍可能尝试加载。

### 9.3 用户禁用

位置：

- `tools/skills_tool.py:_get_disabled_skill_names`
- `tools/skills_tool.py:_is_skill_disabled`

配置来自：

```yaml
skills:
  disabled:
    - some-skill
  platform_disabled:
    telegram:
      - some-skill
```

`scan_skill_commands()` 会跳过禁用 skill。

`skill_view()` 也会检查禁用状态，防止模型直接读取禁用 skill。

---

## 10. Slash autocomplete 如何知道 skill

补全系统在：

- `hermes_cli/commands.py:SlashCommandCompleter`

它接收：

```python
skill_commands_provider=lambda: get_skill_commands()
```

然后在 `get_completions()` 里混合展示：

- 内置 slash commands
- skill bundles
- skill commands
- plugin commands

skill command 的 display meta 来自 skill description：

```python
for cmd, info in self._iter_skill_commands().items():
    cmd_name = cmd[1:]
    if cmd_name.startswith(word):
        description = str(info.get("description", "Skill command"))
        ...
```

所以用户输入 `/git...` 时，能看到相关 skill command，是因为：

```text
scan_skill_commands()
  -> get_skill_commands()
  -> SlashCommandCompleter
  -> completion menu
```

---

## 11. Skill 选择依据到底是什么

总结一下，项目里“选择哪个 skill”的依据分场景。

### 11.1 用户显式输入 slash command

选择依据是命令名：

```text
/github-pr-workflow
  -> scan_skill_commands 里对应的 skill
```

这是确定性的。

### 11.2 启动参数指定

选择依据是传入的 skill identifier：

```text
hermes chat --skills github-pr-workflow
```

也是确定性的。

### 11.3 Gateway channel/topic 绑定

选择依据是 event 上的 `auto_skill`：

```text
event.auto_skill
  -> _load_skill_payload(auto_skill)
```

也是确定性的。

### 11.4 模型自主发现

选择依据不是代码规则，而是模型判断：

```text
用户任务
  -> 模型决定是否调用 skills_list
  -> 根据 name / description / category 选择候选
  -> 调 skill_view 读取完整内容
  -> 按 skill 指令执行
```

这部分是 LLM 决策，不是 Python 代码里硬编码的分类器。

---

## 12. 关键文件索引

建议按下面顺序读：

1. `toolsets.py`
   - 看 skill tools 为什么默认可见
2. `tools/skills_tool.py`
   - 看 `skills_list` / `skill_view` 如何发现和加载 skill
3. `agent/skill_commands.py`
   - 看 slash skill 如何扫描、缓存、构造注入消息
4. `cli.py`
   - 看 CLI 如何 dispatch `/skill-name`
5. `tui_gateway/server.py`
   - 看 TUI / Desktop 的 `commands.catalog`、`complete.slash`、`command.dispatch`
6. `gateway/run.py`
   - 看 messaging channel/topic 的 auto skill 注入
7. `hermes_cli/commands.py`
   - 看 slash autocomplete 如何混入 skill commands

---

## 13. 一句话总结

Hermes 判断是否使用 skill 的核心机制是：

```text
用户/配置显式指定
  -> 精确加载对应 skill

模型认为需要额外工作流知识
  -> 调 skills_list 查看目录
  -> 调 skill_view 读取选中的 skill
```

所以它的设计不是“核心规则自动匹配 skill”，而是：

> 把 skill 做成可发现、可显式调用、可渐进读取的知识/工作流资源，让用户和模型分别在合适时机加载。
