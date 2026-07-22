# Hermes Agent DOCX 读取机制源码导读

这份文档解释 Hermes Agent 为什么能“自动读取 `.docx` 文件”，以及这件事到底是通过 skill、插件，还是核心工具实现的。

先给结论：

> `.docx` 自动读取不是 skill，也不是插件。  
> 它是 Hermes core file tool 的能力，具体由 `read_file` 工具和 `tools/read_extract.py` 里的结构化文档抽取器实现。

---

## 1. 总体链路

`.docx` 文件被读取时，大致链路是：

```text
用户给出 / 上传 / 拖入 .docx
  -> 前端把文件路径或 @file: 引用放进 prompt
  -> agent 看到文件路径/附件提示
  -> 调用 core tool: read_file
  -> read_file 检测到 .docx
  -> tools.read_extract.extract_document_text()
  -> 从 DOCX 内部 XML 抽取文本
  -> 返回带行号的可读文本给模型
```

所以 `.docx` 的读取能力属于：

```text
core tool
  -> file toolset
  -> read_file
  -> read_extract
```

不是：

```text
skill
plugin
model provider
外部 MCP server
```

---

## 2. 为什么模型能调用 read_file

`read_file` 是 Hermes 的核心工具之一。

位置：

- `toolsets.py`

核心工具列表里包含：

```python
_HERMES_CORE_TOOLS = [
    ...
    "read_file", "write_file", "patch", "search_files",
    ...
]
```

这意味着普通 agent session 默认就能看到 `read_file`。

`toolsets.py` 里也定义了 `file` toolset：

```python
"file": {
    "description": "File manipulation tools: read, write, patch (with fuzzy matching), and search (content + files)",
    "tools": ["read_file", "write_file", "patch", "search_files"],
    "includes": []
}
```

所以 `.docx` 能被读取的第一层原因是：

```text
read_file 默认在模型可见的工具 schema 里
```

---

## 3. read_file 的 schema 明确告诉模型支持 DOCX

位置：

- `tools/file_tools.py`

`READ_FILE_SCHEMA` 的 description 里明确写了：

```python
READ_FILE_SCHEMA = {
    "name": "read_file",
    "description": (
        "Read a text file with line numbers and pagination. "
        ...
        "Jupyter notebooks (.ipynb), Word documents (.docx), "
        "and Excel workbooks (.xlsx) are auto-extracted to readable text. "
        ...
    ),
}
```

这很关键：

- 模型看到 `.docx` 路径时，知道可以尝试 `read_file`
- 模型不需要先加载某个 document skill
- 也不需要知道底层如何解压 OOXML

工具描述本身就是给模型的能力提示。

---

## 4. read_file 里如何识别结构化文档

核心入口：

- `tools/file_tools.py:read_file_tool`

`read_file_tool()` 读取文件时，会在普通 binary guard 之前先尝试结构化文档抽取：

```python
from tools.read_extract import (
    ExtractionError,
    extract_document_text,
    is_extractable_document,
)

if is_extractable_document(str(_resolved)):
    try:
        extracted_text = extract_document_text(str(_resolved))
    except ExtractionError:
        logger.debug("document extraction failed for %s", path, exc_info=True)
    else:
        ...
        return json.dumps(result_dict, ensure_ascii=False)
```

这里的顺序很重要：

```text
先尝试 structured-document extraction
再做 binary-extension guard
```

因为 `.docx` 本质上是 zip 包，普通二进制判断会把它当 binary。

如果不先抽取，`.docx` 会被 binary guard 拦掉。

---

## 5. 支持哪些结构化文档

真正的抽取器在：

- `tools/read_extract.py`

支持格式写死在：

```python
EXTRACTABLE_EXTENSIONS = frozenset({".ipynb", ".docx", ".xlsx"})
```

入口函数：

```python
def is_extractable_document(path: str) -> bool:
    return bool(_extension(path))

def extract_document_text(path: str) -> str:
    ext = _extension(path)
    if ext == ".ipynb":
        return _extract_notebook(path)
    if ext == ".docx":
        return _extract_docx(path)
    if ext == ".xlsx":
        return _extract_xlsx(path)
    raise ExtractionError(...)
```

所以当前这套内置抽取支持：

- `.ipynb`
- `.docx`
- `.xlsx`

不包括 `.pdf`。PDF 通常走别的路径，例如渲染、视觉、终端工具或专门的 PDF 处理能力。

---

## 6. DOCX 是怎么被解析的

`.docx` 实际是一个 zip 包，里面包含 XML 文件。

Hermes 的实现没有依赖 `python-docx`、`mammoth`、`pandoc` 之类第三方库，而是用 Python 标准库：

- `zipfile`
- `xml.etree.ElementTree`

核心函数：

```python
def _extract_docx(path: str) -> str:
    try:
        with zipfile.ZipFile(path) as zf:
            root = _zip_xml(zf, "word/document.xml")
    except zipfile.BadZipFile as exc:
        raise ExtractionError(f"Not a valid DOCX: {exc}") from exc
    except OSError as exc:
        raise ExtractionError(str(exc)) from exc

    w = f"{{{_NS_W}}}"
    lines: list[str] = []
    for para in root.iter(f"{w}p"):
        buf: list[str] = []
        for node in para.iter():
            if node.tag == f"{w}t":
                buf.append(node.text or "")
            elif node.tag == f"{w}tab":
                buf.append("\t")
            elif node.tag in {f"{w}br", f"{w}cr"}:
                buf.append("\n")
        lines.extend("".join(buf).split("\n"))
    if not any(line.strip() for line in lines):
        raise ExtractionError("DOCX contains no extractable text")
    return "\n".join(lines).rstrip("\n") + "\n"
```

它做的事情是：

```text
打开 .docx zip
  -> 读取 word/document.xml
  -> 遍历 Word 段落节点 w:p
  -> 收集文本节点 w:t
  -> 处理 tab / 换行
  -> 拼成普通文本
```

这是一种轻量文本抽取，不是完整 Word 排版渲染。

它能很好处理：

- 普通段落
- run 里的文本
- tab
- line break

但它不是完整 Office 引擎，不负责完整还原：

- 样式
- 页眉页脚
- 批注
- 复杂表格布局
- 图片
- 修订痕迹

---

## 7. 抽取后如何返回给模型

`read_file_tool()` 拿到 `extracted_text` 后，会把它按普通文本文件一样分页：

```python
lines = extracted_text.splitlines()
total_lines = len(lines)
end_line = offset + limit - 1
page_text = "\n".join(lines[offset - 1:end_line])
```

然后加行号：

```python
"content": file_ops._add_line_numbers(page_text, offset)
```

返回 JSON：

```json
{
  "content": "1|...",
  "total_lines": 123,
  "file_size": 45678,
  "truncated": false,
  "extracted_document": true
}
```

其中：

```text
"extracted_document": true
```

用来告诉模型：这不是原始文本文件，而是从结构化文档里抽出来的文本。

如果内容很长，会返回分页提示：

```text
Use offset=N to continue reading
```

这和普通大文本文件的读取体验保持一致。

---

## 8. 如果 DOCX 损坏会怎样

如果 `.docx` 看起来是 `.docx`，但不是合法 zip 或缺少 `word/document.xml`：

```python
except ExtractionError:
    logger.debug("document extraction failed for %s", path, exc_info=True)
```

`read_file_tool()` 不会崩溃，而是 fall through 到后续逻辑。

后续 binary guard 会返回类似：

```text
Cannot read binary file 'bad.docx' (.docx).
Use vision_analyze for images, or terminal to inspect binary files.
```

对应测试在：

- `tests/tools/test_read_extract.py`

测试名：

```python
def test_corrupt_docx_falls_through_to_binary_guard(self):
```

说明这是有明确回归测试覆盖的行为。

---

## 9. 用户上传 / 拖入 DOCX 后路径怎么进入 agent

不同前端处理方式不完全一样，但目标都是让 agent 拿到一个可读路径。

### 9.1 CLI 文件拖入

位置：

- `cli.py`

相关函数：

- `_resolve_attachment_path`
- `_detect_file_drop`

CLI 会检测用户输入开头是不是一个真实本地文件路径：

```text
/path/to/report.docx summarize this
```

如果匹配到文件，就把它作为用户附加文件提示进入对话。

### 9.2 Desktop / TUI 非图片附件

位置：

- `tui_gateway/server.py:file.attach`

非图片文件附件会被 staging 到 session workspace：

```text
.hermes/desktop-attachments/
```

然后返回：

```text
@file:<path>
```

这让前端可以把附件作为 `@file:` 引用拼进用户消息。

### 9.3 远程 gateway 场景

`file.attach` 还处理一个重要问题：

```text
用户的文件在客户端机器上
但 agent 运行在 gateway / remote process 上
```

代码会：

1. 尝试解析原始路径
2. 如果 gateway 看不到这个路径，就用 `data_url` 上传的 bytes
3. 写入 `.hermes/desktop-attachments/`
4. 返回 gateway 可见路径

这样 agent 后续调用 `read_file` 时，读的是 gateway 进程能访问到的真实文件。

---

## 10. @file: 引用和 DOCX 的关系

`@file:` 引用处理在：

- `agent/context_references.py`

CLI 在发送消息前会调用：

```python
preprocess_context_references(...)
```

位置：

- `cli.py`

逻辑大意：

```text
用户消息里有 @file:xxx
  -> preprocess_context_references
  -> 尝试展开引用
```

但 `.docx` 这类文件通常会被 `context_references` 视为 binary，不直接 inline 成文本。

`_binary_reference_block()` 会生成一段提示：

```text
binary file, not inlined as text.
It is available on disk at `<path>`.
Use your tools to work with it ...
```

这不是失败，而是引导模型下一步调用工具。

随后模型通常会调用：

```text
read_file(path="<that docx path>")
```

然后触发 `read_file` 的 `.docx` 自动抽取。

所以 `@file:` 和 `.docx` 的关系是：

```text
@file:
  -> 把文件位置告诉模型
  -> 不一定直接展开 DOCX 内容
  -> 模型再用 read_file 真正抽取文本
```

---

## 11. 这和 skill 的关系

skill 不负责 `.docx` 的底层读取。

skill 可能做的是：

- 告诉模型遇到合同/报告/简历时应该怎么分析
- 提供文档处理工作流
- 指导输出格式
- 调用脚本进一步处理文档

但即使没有任何相关 skill，只要 `read_file` 可用，模型仍然能读取 `.docx` 的文本。

所以准确分层是：

```text
read_file / read_extract
  -> 提供底层读取能力

skill
  -> 可选的任务流程指导

plugin
  -> 可选扩展，不是当前 DOCX 读取的默认路径
```

---

## 12. 这和插件的关系

插件也不是默认 `.docx` 读取能力的来源。

虽然插件可以注册工具，理论上也可以提供更强的 Office/PDF/文档处理能力，但当前 `.docx` 的基础读取来自内置文件工具。

判断依据是：

- `read_file` 在 `_HERMES_CORE_TOOLS` 中
- `.docx` 抽取代码在 `tools/read_extract.py`
- `tools/read_extract.py` 只被 `tools/file_tools.py` 调用
- 没有通过 plugin discovery 注册 `.docx` reader

---

## 13. 测试覆盖

相关测试在：

- `tests/tools/test_read_extract.py`

覆盖点包括：

- `.docx` 扩展名识别
- 普通段落和 runs 抽取
- tab 和 break 抽取
- 非 zip 的坏 `.docx` 报 `ExtractionError`
- 缺少 `word/document.xml` 报 `ExtractionError`
- `read_file_tool()` 集成测试
- 损坏 `.docx` fall through 到 binary guard

其中集成测试：

```python
def test_docx_read_extracts(self):
    ...
    res = json.loads(read_file_tool(p))
    self.assertTrue(res.get("extracted_document"))
    self.assertIn("Report body", res["content"])
```

这说明 `.docx` 支持不是偶然行为，而是 `read_file` 工具的正式能力。

---

## 14. 关键文件索引

建议按下面顺序读：

1. `toolsets.py`
   - 看 `read_file` 为什么是 core tool
2. `tools/file_tools.py`
   - 看 `READ_FILE_SCHEMA` 和 `read_file_tool`
3. `tools/read_extract.py`
   - 看 `.docx` / `.xlsx` / `.ipynb` 的实际抽取逻辑
4. `agent/context_references.py`
   - 看 `@file:` 引用如何处理 binary 文件
5. `tui_gateway/server.py`
   - 看 Desktop/TUI 文件附件如何 staging
6. `cli.py`
   - 看本地文件拖入和 `@file:` 预处理
7. `tests/tools/test_read_extract.py`
   - 看 `.docx` 行为的测试边界

---

## 15. 一句话总结

Hermes 能自动读取 `.docx`，是因为：

```text
read_file 是 core tool
  -> read_file 内置结构化文档抽取
  -> .docx 由 tools/read_extract.py 用 stdlib 解 zip + XML
  -> 抽取结果按普通文本分页返回给模型
```

skill 和 plugin 在这里都不是必要条件。
