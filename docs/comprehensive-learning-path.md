# 完整学习路径总览

## 项目架构

```
cli.ts / __main__.py        ← 入口、参数解析、REPL
        ↓
agent.ts / agent.py          ← 核心引擎：主循环、双后端、权限、压缩
   ↓         ↓         ↓
prompt.ts   memory.ts  skills.ts    ← 系统提示构造、记忆召回、技能解析
prompt.py   memory.py  skills.py
   ↓         ↓         ↓
subagent.ts session.ts mcp.ts       ← 子代理、会话持久化、MCP协议
subagent.py session.py mcp_client.py
        ↓
tools.ts / tools.py          ← 13个工具定义与执行、权限校验
ui.ts / ui.py                ← 终端UI（spinner、diff高亮、图标）
frontmatter.ts / frontmatter.py  ← YAML frontmatter解析
```

## 10层学习递进

### 第1层：入口与配置

**CLI参数（11个flags）**：`--yolo`(bypassPermissions), `--plan`(plan模式), `--accept-edits`, `--dont-ask`, `--thinking`(扩展思考), `--model/-m`, `--api-base`(OpenAI兼容端点), `--resume`(恢复会话), `--max-cost`(美元预算), `--max-turns`(轮次限制), `--help`

**API配置优先级**：
```
OPENAI_API_KEY + OPENAI_BASE_URL  →  OpenAI格式
    ↓ (未设置)
ANTHROPIC_API_KEY (+ ANTHROPIC_BASE_URL)  →  Anthropic格式
    ↓ (未设置)
OPENAI_API_KEY  →  OpenAI格式 (fallback key)
```

**REPL命令**：`/clear`(清历史), `/plan`(切换plan模式), `/cost`(显示用量), `/compact`(手动压缩), `/memory`(列出记忆), `/skills`(列出技能), `/<skill-name>`(调用技能)

**双重Ctrl+C**：第一次=abort当前处理, 第二次=warn then exit

### 第2层：Agent主循环

核心模式：
```
while(true):
    runCompressionPipeline()     // 压缩管道 (Tier 1-3, 零API成本)
    消费memoryPrefetch (非阻塞)   // 记忆注入
    startSpinner()
    response = callModelStream()  // 流式API调用
    stopSpinner()
    更新token计数
    if 无tool_use: break          // 退出条件
    检查budget(turns/cost)
    for each tool_use:
        权限检查 → 执行工具 → 收集结果
    将tool_results推入消息历史
```

### 第3层：双后端流式调用

**Anthropic流式** (SDK `messages.stream()`)：
```
stream.on("content_block_start") → 记录tool_use index, id, name
stream.on("content_block_delta") → 累积partial_json到对应index
stream.on("content_block_stop") → parse JSON → fire onToolBlockComplete回调
finalMessage.content = 过滤掉thinking blocks
```

**OpenAI流式** (手动chunk累积)：
```
for await (chunk of stream):
    delta.content? → 输出文本
    delta.tool_calls? → Map<index, {id, name, arguments}>
        // arguments分片到达, 字符串拼接
chunk.usage? → 记录token
最后: 重建ChatCompletion {choices: [{message: {tool_calls: [...]}}]}
```

### 第4层：流式工具执行

**CONCURRENCY_SAFE_TOOLS** = `{read_file, list_files, grep_search, web_fetch}`

**Anthropic**: `content_block_stop`触发时立即`earlyExecutions.set(id, executeToolCall(...))` — 工具在模型还在生成时并行运行

**OpenAI**: 两阶段 —
- Phase 1: 串行解析JSON + 权限检查 (需要用户交互)
- Phase 2: 连续安全工具分组 → `Promise.all` / `asyncio.gather` 并发

### 第5层：权限系统 (4层防御)

```
Layer 1: 模式检查 (default/plan/acceptEdits/bypassPermissions/dontAsk)
Layer 2: 配置规则 (settings.json deny优先, 支持tool("pattern*")前缀匹配)
Layer 3: 内建危险检测 (16个正则: rm, git push/reset/clean, sudo, mkfs, dd, kill...)
Layer 4: 会话白名单 (_confirmed_paths, 一次确认后记住)
```

### 第6层：上下文压缩 (4层管道)

| Tier | 名称 | 触发条件 | 操作 |
|------|------|---------|------|
| 1 | Budget | 利用率>50% | 截断大型tool_result (15K/30K预算) |
| 2 | Snip | 利用率>60% | 同文件去重(留最近3次), 占位符替换 |
| 3 | Microcompact | 空闲>5分钟 | 清除旧结果为"[Old result cleared]" |
| 4 | Auto-compact | 利用率>85% | LLM摘要压缩 → 2条消息替代整段历史 |

### 第7层：Memory系统

4种类型：`user / feedback / project / reference`
文件格式：`{memory_dir}/{type}_{slug}.md` (YAML frontmatter + Markdown body)
索引：`MEMORY.md` (200行/25KB截断)

**3个门控**：
- `isQuerySubstantial()`: TS额外检查CJK字符≥2, Python仅检查多词
- Session预算: 60KB累积
- Memory文件必须存在

**语义召回流程**：
1. `scanMemoryHeaders()` — 只读前30行frontmatter (轻量)
2. 格式化manifest → sideQuery模型调用
3. 模型选择最多5个 → 读取完整文件
4. 非阻塞polling: `settled`/`consumed` flags
5. `alreadySurfaced` Set防止重复, 1天新鲜度警告

### 第8层：Skills系统

`SKILL.md` frontmatter元数据：`name, description, when_to_use, allowed-tools, user-invocable, context`

**双调用方式**：CLI `/skill-name` 或模型 `skill` tool
**双执行模式**：Inline (注入prompt) / Fork (隔离sub-agent, runOnce)
**模板替换**：`$ARGUMENTS`, `${ARGUMENTS}`, `${CLAUDE_SKILL_DIR}`
**去重**：2来源 (user `~/.claude/skills/` + project `.claude/skills/`), Map, project优先

### 第9层：Plan Mode

`enter_plan_mode` / `exit_plan_mode` 是 **deferred tools** — schema不发往API，`tool_search`激活

- **进入**：切换到只读 + 生成plan文件路径 + 注入Plan Mode提示
- **退出4选择**：
  1. clear-and-execute: 清上下文 + 自动接受编辑
  2. execute: 保留上下文 + 自动接受编辑
  3. manual-execute: 保留上下文 + 每次确认编辑
  4. keep-planning: 拒绝 + 反馈修改

### 第10层：Multi-Agent 与 MCP

**3种内建Agent**：
- `explore`: 只读 (read_file, list_files, grep_search)
- `plan`: 只读 + 结构化计划
- `general`: 全部工具除去agent tool

**自定义Agent**：`~/.claude/agents/*.md` + `.claude/agents/*.md`, frontmatter + allowed-tools

**Fork-Return模式**：`runOnce()` → 子Agent → token合并回父Agent

**MCP协议**：JSON-RPC over stdio → `_read_loop()`逐行读取 → `pending: Map<id, Future>` → 3段命名 `mcp__server__tool` → lazy init + 15s超时

