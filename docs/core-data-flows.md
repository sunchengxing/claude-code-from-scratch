# 核心数据流详解

## 1. 主循环数据流 (Agent Loop)

```
用户输入 "fix the bug in login.ts"
  │
  ▼
agent.chat(userMessage)
  │
  ├─ push {role:"user", content:"fix..."} 到消息历史
  ├─ checkAndCompact()  // 85%窗口→自动压缩
  ├─ startMemoryPrefetch() // 异步非阻塞
  │
  ▼
while(true):
  │
  ├─ runCompressionPipeline()
  │   ├─ Tier1: budgetToolResults (截断大结果)
  │   ├─ Tier2: snipStaleResults (去重同文件读取)
  │   └─ Tier3: microcompact (清除旧结果)
  │
  ├─ 消费 memoryPrefetch (poll settled/consumed)
  │   └─ 注入到last user message (避免连续user消息)
  │
  ├─ response = callStream(onToolBlockComplete)
  │   │
  │   ├─ 流式输出文本给用户
  │   ├─ 追踪 tool_use blocks (index → {id, name, inputJson})
  │   ├─ content_block_stop → onToolBlockComplete()
  │   │   └─ 若安全工具 → earlyExecutions.set(id, executeNow)
  │   └─ 过滤thinking blocks → 返回finalMessage
  │
  ├─ push {role:"assistant", content:[...blocks]}  // 包括tool_use
  │
  ├─ if 无tool_use → break (对话结束)
  │
  ├─ 检查budget (cost/turns)
  │
  └─ 处理每个tool_use:
      ├─ 如果已在earlyExecutions中 → 直接await结果
      ├─ 否则 → 权限检查 → 执行 → persistLargeResult
      └─ push {role:"user", content:[tool_results]}
```

## 2. Anthropic流式数据流

```
SDK stream事件 → 本地追踪
═══════════════════════════════════════════

content_block_start (index=0, type="text")
  │
content_block_delta (index=0, delta.text="I'll ")
  │   └→ emitText("I'll ")  // 实时输出
  │
content_block_start (index=1, type="tool_use", id="toolu_01", name="read_file")
  │   └→ toolBlocksByIndex.set(1, {id:"toolu_01", name:"read_file", inputJson:""})
  │
content_block_delta (index=1, delta.type="input_json_delta", partial_json='{"file')
  │   └→ toolBlocksByIndex[1].inputJson += '{"file'
  │
content_block_delta (index=1, partial_json='_path":')
  │   └→ toolBlocksByIndex[1].inputJson += '_path":'
  │
content_block_delta (index=1, partial_json='"src/login.ts"}')
  │   └→ toolBlocksByIndex[1].inputJson += '"src/login.ts"}'
  │
content_block_stop (index=1)
  │   └→ parse inputJson → {file_path: "src/login.ts"}
  │   └→ onToolBlockComplete({type:"tool_use", id:"toolu_01", name:"read_file", input:{...}})
  │       └→ name in CONCURRENCY_SAFE_TOOLS ✓
  │       └→ checkPermission("read_file", input, mode) → {action:"allow"}
  │       └→ earlyExecutions.set("toolu_01", executeToolCall("read_file", input))
  │           // 工具开始在后台执行！模型可能还在生成
  │
stream.finalMessage()
  │   └→ filter out thinking blocks
  │   └→ return {content: [text_block, tool_use_block], usage: {input_tokens, output_tokens}}
  │
  ▼
处理tool_use: earlyExecutions.get("toolu_01") → await → 拿到结果
```

## 3. OpenAI流式数据流

```
for await (chunk of stream):
  │
  ├─ chunk.choices[0].delta.content === "I'll "
  │   └→ emitText("I'll ")
  │
  ├─ chunk.choices[0].delta.tool_calls === [
  │     {index: 0, id: "call_01", function: {name: "read_", arguments: ""}}
  │   ]
  │   └→ toolCalls.set(0, {id:"call_01", name:"read_", arguments:""})
  │
  ├─ chunk.choices[0].delta.tool_calls === [
  │     {index: 0, function: {arguments: 'file'}}
  │   ]
  │   └→ toolCalls[0].arguments += 'file'
  │
  ├─ ... 更多arguments分片累积 ...
  │
  ├─ chunk.usage → {prompt_tokens: 500, completion_tokens: 50}
  │   └→ 记录usage
  │
  └─ chunk.choices[0].finish_reason === "tool_calls"
      └→ 重建ChatCompletion:
          {choices:[{message:{tool_calls:[
            {id:"call_01", function:{name:"read_file", arguments:'{"file_path":"src/login.ts"}'}}
          ]}}]}
```

## 4. 消息交替规则

API要求user/assistant交替，工具结果的处理：

```
Anthropic格式:
  [user] "fix the bug"
  [assistant] {content: [{type:"tool_use", ...}]}
  [user] {content: [{type:"tool_result", tool_use_id:"...", ...}]}  ← 一个user消息含多个result block
  [assistant] {content: [{type:"text", ...}, {type:"tool_use", ...}]}

OpenAI格式:
  [user] "fix the bug"
  [assistant] {tool_calls: [...]}
  [tool] {tool_call_id:"call_01", content:"..."}  ← 每个工具调用一个tool消息
  [tool] {tool_call_id:"call_02", content:"..."}
  [assistant] {content:"...", tool_calls:[...]}
```

**Memory注入的特殊处理**：记忆必须追加到已有的user消息中（而非新建user消息），避免连续两个user消息破坏交替规则。
