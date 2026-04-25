# Evidence sample (b):Disallowed MCP tool 的 wire 形态

**采集日期**:2026-04-25
**Claude Code 版本**:2.1.119
**关联 issue**:M1 #1 / [open-trad/opentrad#18](https://github.com/open-trad/opentrad/issues/18)
**为后续准备**:M1 #11 / [open-trad/opentrad#28](https://github.com/open-trad/opentrad/issues/28) Risk Gate(deny 路径事件流形态参考)

## 用途

记录 CC 尝试调用一个**没在 `--allowedTools` 里的** MCP tool 时,wire 流如何表达"被拒绝"。供 M1 #28 实施 RiskGate `deny` 路径时,理解 CC 收到 deny 后的事件流形态——尤其 user tool_result 的 `is_error: true` 形态,以及 result 事件的 `permission_denials` 数组结构。

> ⚠️ **重要**:本样本采集的是**CC 内建权限机制拒绝 tool 调用**的路径(因 tool 不在 `--allowedTools` 中,被 CC 客户端层拦截)。M1 #28 RiskGate 的 deny 路径**走的是另一条**——RiskGate 在 mcp-server 端做拦截,返回 MCP error。两者 wire 形态不完全相同(本样本是参考性质,验证 deny 后 CC 的恢复行为模式;真实 RiskGate deny 形态需 M1 #28 实施时再抓一次样本)。

## 采集环境

- **第三方 MCP server**:`@modelcontextprotocol/server-everything`(同 sample a)
- **mcp-config**:同 sample a 的 `/tmp/m1-mcp-everything.json`
- **CC 命令**(关键差异:`--allowedTools` 故意不含 echo):

```bash
claude -p \
  --output-format stream-json --verbose \
  --session-id 22222222-2222-2222-2222-222222222222 \
  --allowedTools "mcp__everything__add" \
  --mcp-config /tmp/m1-mcp-everything.json --strict-mcp-config \
  "Use the echo tool from the everything MCP server to echo back the message 'OpenTrad evidence sample B'. Use only that one tool, do not use any other tool."
```

- **结果**:exit 0,11 行 NDJSON,duration 17.043s,total cost $0.170280,**`permission_denials` 非空**

## Redact 范围

同 sample a:

- `cwd`:用户名替换为 `<USER>`
- `memory_paths`:整字段已删
- `plugins`:整字段已删

其他保留(包括 session_id 占位 UUID、内部 uuid、tool_use_id、错误消息文本等)。

## 完整事件流(11 行 NDJSON)

### 行 1:`system/init`

```ndjson
{"type":"system","subtype":"init","cwd":"/Users/<USER>/Desktop/open-trad/opentrad","session_id":"22222222-2222-2222-2222-222222222222","tools":[...,"mcp__everything__echo","mcp__everything__get-annotated-message",...],"mcp_servers":[{"name":"everything","status":"connected"}],...,"claude_code_version":"2.1.119",...,"uuid":"5ec293ef-5669-4972-9d18-f6f6498f4182","fast_mode_state":"off"}
```

> 💡 **`tools` 列表里仍含 `mcp__everything__echo`**:CC 拉起 server-everything 后能看到所有 tools。`--allowedTools` 是一个**白名单过滤器**,CC 看到完整列表但只允许调用白名单内的。**M1 #28 RiskGate 设计**:OpenTrad 的 RiskGate 拦截发生在 mcp-server middleware 端(详见 03 §4.5),wire 流形态会与本样本不同——OpenTrad RiskGate deny 时返回的是结构化 MCP error,而 CC 内建拦截返回的是 string content 错误消息(见行 9)。

### 行 2:`rate_limit_event`

```ndjson
{"type":"rate_limit_event","rate_limit_info":{"status":"allowed","resetsAt":1777118400,"rateLimitType":"five_hour","overageStatus":"rejected","overageDisabledReason":"out_of_credits","isUsingOverage":false},"uuid":"9b79685f-c740-4495-8709-a44ba1b0034c","session_id":"22222222-2222-2222-2222-222222222222"}
```

### 行 3:`assistant`(thinking,LLM 第一轮思考)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01FbmvovvhtdRKhhCHJGGZmd",...,"content":[{"type":"thinking","thinking":"","signature":"EpcOCl..."}],"stop_reason":null,...},...,"uuid":"f3968512-d76f-4732-a809-561d89285c0d"}
```

### 行 4:`assistant`(text,中间解释)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01FbmvovvhtdRKhhCHJGGZmd",...,"content":[{"type":"text","text":"需要先加载 echo 工具的 schema 才能调用它。"}],...},...,"uuid":"1aa15339-e3a3-41eb-939a-1febfd919af6"}
```

> 💡 **同 msgId**(`msg_01Fbmv...`)与行 3 共享:LLM 同一轮 generation 内,thinking + text 拆成两个独立 wire events。**印证 per-wire-event 语义**——`isLast=true` 在每个 wire event 内都为真。

### 行 5:`assistant`(tool_use:ToolSearch)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01FbmvovvhtdRKhhCHJGGZmd",...,"content":[{"type":"tool_use","id":"toolu_014RUwFZDSSockm28VfVo5C6","name":"ToolSearch","input":{"query":"select:mcp__everything__echo","max_results":1},"caller":{"type":"direct"}}],...},...,"uuid":"ee78d0b2-2008-4fdb-96f2-8962f837749b"}
```

### 行 6:`user`(tool_result:ToolSearch 返回 echo reference)

```ndjson
{"type":"user","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_014RUwFZDSSockm28VfVo5C6","content":[{"type":"tool_reference","tool_name":"mcp__everything__echo"}]}]},...,"timestamp":"2026-04-25T08:41:49.200Z","tool_use_result":{"matches":["mcp__everything__echo"],"query":"select:mcp__everything__echo","total_deferred_tools":34}}
```

> 💡 **CC 仍然 schedule 了 ToolSearch**——尽管 echo 已经在 `tools` 列表里,LLM 不知道它已经被 disallow。这印证 CC 内建拦截发生在**实际调用时**,不是 tool 列表过滤时。

### 行 7:`assistant`(thinking,第二轮思考)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01RzfvCgyMCiL198HV1GB718",...,"content":[{"type":"thinking","thinking":"","signature":"EtIB..."}],...,"diagnostics":{"cache_miss_reason":{"type":"system_changed","cache_missed_input_tokens":24812}},...},...,"uuid":"bfc690ec-ec45-411e-9e78-a4da3a3e205f"}
```

### 行 8:`assistant`(tool_use:实际尝试调用 echo)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01RzfvCgyMCiL198HV1GB718",...,"content":[{"type":"tool_use","id":"toolu_01TcmMWXRQasDrJd9F8NxxSM","name":"mcp__everything__echo","input":{"message":"OpenTrad evidence sample B"},"caller":{"type":"direct"}}],...},...,"uuid":"bcf6c513-f9b4-4133-aad3-1967c498c1ef"}
```

> 💡 **`tool_use_id: "toolu_01TcmMWXRQasDrJd9F8NxxSM"`**:记住这个 ID,稍后会出现在 result 事件的 `permission_denials` 数组里。

### 行 9:`user`(tool_result:**核心 deny 信号**)

```ndjson
{"type":"user","message":{"role":"user","content":[{"type":"tool_result","content":"Claude requested permissions to use mcp__everything__echo, but you haven't granted it yet.","is_error":true,"tool_use_id":"toolu_01TcmMWXRQasDrJd9F8NxxSM"}]},...,"timestamp":"2026-04-25T08:41:52.391Z","tool_use_result":"Error: Claude requested permissions to use mcp__everything__echo, but you haven't granted it yet."}
```

> 💡 **关键发现:CC 内建拦截的 wire 形态**:
>
> - `tool_result.content` 是**字符串**,不是 array of `{type:"text",text:"..."}`(对比 sample a 行 7 正常 result 的结构)。这是 wire schema 必须用 `.passthrough()` 或 `z.union([string, array])` 兜底的边界 case。
> - `is_error: true` 在 tool_result 顶层(不在 content 里)。
> - `tool_use_id` 重复 wire 行 8 的 `toolu_01Tcm...`,可关联到具体被拒的 tool 调用。
> - `tool_use_result` 字段是字符串("Error: ..."),不是结构化对象——表明这是 CC 客户端层生成的错误信息(非 MCP server 返回)。
>
> 💡 **M1 #28 设计参考**:OpenTrad RiskGate deny 时,从 mcp-server middleware 返回的 MCP error 应当也采用类似形态(`is_error: true` + 人类可读 string content)。但**RiskGate deny 时不应**让 CC 收到"requested permissions but not granted"这种暗示"用户授权后能继续"的措辞——RiskGate 的 deny 是终态决策,应当返回明确的"declined by Risk Gate"消息,避免 CC 死循环重试。

### 行 10:`assistant`(text:最终承认被拦截)

```ndjson
{"type":"assistant","message":{...,"id":"msg_01Aciao7cZfCfrfci7zprdoU",...,"content":[{"type":"text","text":"echo 工具调用被拦截，等待你授权后即可回显 `OpenTrad evidence sample B`。"}],...,"diagnostics":null,...},...,"uuid":"6b6fae09-cd43-4058-b717-fdb4e8583df2"}
```

> 💡 **CC 的恢复行为**:CC 收到 deny 后**没有重试**,而是平静地告知用户"工具被拦截"。这对 RiskGate UX 是好消息——deny 不会触发死循环。**M1 #28 实施时**:OpenTrad RiskGate 的 deny error message 应明确指出原因(用户拒绝 / 规则拒绝 / 超时),让 CC 能给出准确的解释。

### 行 11:`result`(任务结束,**`permission_denials` 数组关键**)

```ndjson
{"type":"result","subtype":"success","is_error":false,...,"duration_ms":17043,"total_cost_usd":0.17028,"stop_reason":"end_turn",...,"modelUsage":{...},"permission_denials":[{"tool_name":"mcp__everything__echo","tool_use_id":"toolu_01TcmMWXRQasDrJd9F8NxxSM","tool_input":{"message":"OpenTrad evidence sample B"}}],"terminal_reason":"completed","fast_mode_state":"off","uuid":"b649e472-00b1-4f3b-ab1e-4992115d16fd"}
```

> 💡 **`permission_denials` 数组**:本次 task 内所有被拦截的 tool 调用都汇总在这里。结构:
> ```typescript
> {
>   tool_name: string;          // 如 "mcp__everything__echo"
>   tool_use_id: string;        // 关联到行 8 的 tool_use 事件
>   tool_input: unknown;        // 被拦截调用时尝试的参数
> }
> ```
>
> **M1 #28 audit log 字段映射参考**:OpenTrad 的 audit_log 表里 deny 记录可以直接映射这三个字段(再加 timestamp / decision / reason 等元数据)。
>
> 💡 **`subtype: "success"` + `is_error: false`**:即使有 permission denial,task 整体仍 success(LLM 完成了"告知用户被拦截"的对话)。
>
> 💡 **`stop_reason: "end_turn"` + `terminal_reason: "completed"`**:正常结束,非 abort / error。

## 关键发现汇总(给 M1 #11 / #28)

| 发现 | 影响 / 用途 |
|---|---|
| `tool_result.content` 在 deny 时是 string,正常时是 array of {type, text} | wire schema 必须容忍两种形态 |
| `is_error: true` 在 tool_result 顶层 | RiskGate middleware 返回的 deny error 模仿这个结构 |
| CC 收到 deny **不重试**(平静告知用户) | RiskGate UX 设计:deny 是终态,不会触发死循环 |
| `result.permission_denials` 数组聚合所有被拦截调用 | M1 #28 audit log 字段映射:`tool_name` / `tool_use_id` / `tool_input` 直接抄 |
| CC 内建拦截的错误消息暗示"用户授权后可继续" | OpenTrad RiskGate deny 应明确说"declined by Risk Gate / by user / by rule",避免歧义 |
| 同一逻辑 message 内 thinking + text + tool_use 拆成 3 个 wire events | 印证 D6 后续 per-wire-event 语义(已落地架构 §4.2 勘误 3) |

## 与本样本不同的 wire 形态(M1 #28 实施时再补抓)

本样本是 **CC 内建权限机制**的 deny 形态。M1 #28 真实场景是 **OpenTrad RiskGate 在 mcp-server middleware 端 deny**,wire 形态预期会是:

- `tool_result.content` 是 mcp-server 返回的结构化 MCP error(可能是 array of `{type:"text",text:"Risk Gate denied: ..."}`)
- `is_error: true` 在 tool_result 顶层(同样)
- `result.permission_denials` **可能为空**(因为 CC 不知道这是"权限拒绝",对 CC 而言是"tool 报错")

M1 #28 实施时建议在 `evidence/m1-impl-samples/` 重新采集一份。
