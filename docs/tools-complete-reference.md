# 13个工具完整参考

## 工具总览

| 工具 | 类型 | 安全 | 延迟 | 关键实现 |
|------|------|------|------|---------|
| read_file | 只读 | ✓ | ✗ | 编号行 + mtime记录 |
| list_files | 只读 | ✓ | ✗ | glob + 200文件上限 |
| grep_search | 只读 | ✓ | ✗ | 系统grep优先 + 100条上限 |
| web_fetch | 只读 | ✓ | ✗ | urllib + HTML剥离 |
| write_file | 写入 | ✗ | ✗ | 自动创建目录 + 30行预览 |
| edit_file | 写入 | ✗ | ✗ | 引号标准化 + diff输出 |
| run_shell | 危险 | ✗ | ✗ | subprocess + 16条危险正则 |
| skill | 特殊 | ✗ | ✗ | inline/fork双模式 |
| agent | 特殊 | ✗ | ✗ | fork-return子代理 |
| enter_plan_mode | 特殊 | ✗ | ✓ | deferred, tool_search激活 |
| exit_plan_mode | 特殊 | ✗ | ✓ | deferred, 4选项审批 |
| tool_search | 特殊 | ✗ | ✗ | 激活deferred工具 |
| mcp__* | 外部 | 按服务器 | ✗ | JSON-RPC over stdio |

## read_file

```python
# 编号行输出: "   1 | content"
f"{i+1:4d} | {line}"
```
**read-before-edit保护**：读取时记录`mtimeMs`到`readFileState: Map<path, mtime>`。write/edit前必须验证已读且mtime未变更（外部修改检测）。

## write_file

```python
# 自动创建目录, 30行预览
os.makedirs(os.path.dirname(file_path), exist_ok=True)
# 返回: "Successfully wrote to {path}\n{preview}"
```
**特殊行为**：写入memory目录时自动重建MEMORY.md索引。

## edit_file

**引号标准化**（解决Unicode引号vs直引号问题）：
```python
SMART_QUOTES = {'‘': "'", '’': "'", '′': "'",
                '“': '"', '”': '"', '″': '"'}
```

**双重匹配检测**：
```python
if content.count(actual) > 1:
    return "Error: old_string matches multiple locations"
```

**匹配策略**：精确匹配 → 引号标准化重试 → 返回`"matched via quote normalization"`

**Diff输出**：`@@ -line,col +line,col @@` 标记，`-`行红色，`+`行绿色

## grep_search

**双引擎**：
1. `grep --line-number --color=never -r <pattern> <path>` (优先)
2. Python `re.compile(pattern).search(line)` (fallback, 100条匹配上限)

## run_shell

**危险命令检测（16条正则）**：
```
rm -rf, git push --force, git reset --hard, git clean,
sudo, mkfs, dd, >/dev/sda, kill -9, pkill, reboot,
shutdown, del /f, rmdir /s, format, taskkill /f,
Remove-Item -Force, Stop-Process -Force
```

**执行**：`subprocess.run(command, shell=True, timeout=30, capture_output=True)` — 5MB buffer，不追踪

## web_fetch

```python
req = urllib.request.Request(url, headers={"User-Agent": "MiniClaude/1.0"})
html = urllib.request.urlopen(req, timeout=15).read().decode()
# HTML剥离: script/style标签移除 → 所有标签移除 → 实体解码
```

## 大结果持久化

```python
# 超过30KB → 写入 ~/.mini-claude/tool-results/{timestamp}-{tool}.txt
# 返回: 200行预览 + 文件路径
```

## Deferred Tools (enter_plan_mode, exit_plan_mode)

- Schema不从`getActiveToolDefinitions()`返回
- `tool_search`激活 → 加入`_activated_tools` set
- `getActiveToolDefinitions()`检查`_activated_tools`来决定是否返回完整schema

## MCP工具命名

`mcp__{serverName}__{toolName}` — 3段式，`parts.slice(2).join("__")`处理toolName中的`__`
