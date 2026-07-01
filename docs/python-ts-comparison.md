# Python ↔ TypeScript 实现对比

## 异步模型对比

| 概念 | Python | TypeScript |
|------|--------|------------|
| 异步原语 | `asyncio` | Promise (microtask queue) |
| 取消操作 | `asyncio.Task.cancel()` | `AbortController.abort()` |
| Future/Promise 等待 | `await future` | `await promise` |
| 并发执行 | `asyncio.gather(*tasks)` | `Promise.all(promises)` |
| 后台任务 | `asyncio.create_task(coro)` | `new Promise(...)` (fire-and-forget) |
| 流式迭代 | `async for event in stream:` | `for await (const chunk of stream)` |
| SDK流 | `client.messages.stream()` | `client.messages.stream()` |
| 事件类型 | `event.type` (字符串属性) | `event.type` (字符串属性) |

**earlyExecutions对比**：
```python
# Python: dict[str, asyncio.Task]
early_executions: dict[str, asyncio.Task] = {}
task = asyncio.create_task(self._execute_tool_call(...))
early_executions[block["id"]] = task
```
```typescript
// TypeScript: Map<string, Promise<string>>
const earlyExecutions = new Map<string, Promise<string>>();
earlyExecutions.set(block.id, this.executeToolCall(block.name, input));
```

## Spinner对比

| 项目 | Python | TypeScript |
|------|--------|------------|
| 帧集 | braille `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` | 同 |
| 定时器 | `threading.Thread` (daemon) | `setInterval` (80ms) |
| 停止 | `Event.set()` + `join(timeout=1)` | `clearInterval` + `\r\x1b[K` |

## isQuerySubstantial差异

```python
# Python: 仅检查空格分隔的多词
def _is_query_substantial(query: str) -> bool:
    return bool(re.search(r"\s", query.strip()))
```
```typescript
// TypeScript: 额外检查CJK字符
function isQuerySubstantial(query: string): boolean {
  const cjkRegex = /[一-鿿぀-ヿ가-힯]/g;
  const cjkMatches = trimmed.match(cjkRegex);
  if (cjkMatches && cjkMatches.length >= 2) return true;
  if (/\s/.test(trimmed)) return true;
  return false;
}
```

## 项目哈希

```python
# Python
sha256(cwd.encode()).hexdigest()[:16]
```
```typescript
// TypeScript
createHash("sha256").update(process.cwd()).digest("hex").slice(0, 16)
```

## 消息格式差异

**Anthropic**: Python用`dict`，TypeScript用interface类型
```python
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "...", "content": "..."}]}
```
```typescript
{ role: "user", content: [{ type: "tool_result", tool_use_id: "...", content: "..." }] }
```

**OpenAI**: Python用`dict`，TypeScript用interface类型
```python
{"role": "tool", "tool_call_id": "call_01", "content": "result"}
```
```typescript
{ role: "tool", tool_call_id: "call_01", content: "result" }
```

## 文件组织差异

Python用`__init__.py`导出公共API，TypeScript用`export`关键字 + `package.json`的`"main"`字段。

Python`__main__.py`模块自动成为`python -m mini_claude`的入口，TypeScript用`#!/usr/bin/env node` + `bin`字段。

## 权限确认UI

Python版提供`confirm_fn`回调参数（可选外部confirm函数），TypeScript版提供`setConfirmFn()`方法在REPL构建时注入readline实例——两者都解决同一问题：避免创建第二个readline实例导致第一个被关闭。
