# Evidence sample (a):MCP tool 调用的 wire 形态

**采集日期**:2026-04-25
**Claude Code 版本**:2.1.119
**关联 issue**:M1 #1 / [open-trad/opentrad#18](https://github.com/open-trad/opentrad/issues/18)
**为后续准备**:M1 #8 / [open-trad/opentrad#25](https://github.com/open-trad/opentrad/issues/25) MCP server + IPC bridge

## 用途

记录 CC 通过 MCP server 调用工具时的完整 stream-json 事件流。供 M1 #25 实施 OpenTrad 自己的 MCP server 时参考 wire 协议形态。

## 采集环境

- **第三方 MCP server**:`@modelcontextprotocol/server-everything`(stdio transport,通过 npx 拉起)
- **mcp-config**(写到 `/tmp/m1-mcp-everything.json`):

```json
{
  "mcpServers": {
    "everything": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-everything"]
    }
  }
}
```

- **CC 命令**:

```bash
claude -p \
  --output-format stream-json --verbose \
  --session-id 11111111-1111-1111-1111-111111111111 \
  --allowedTools "mcp__everything__echo" \
  --mcp-config /tmp/m1-mcp-everything.json --strict-mcp-config \
  "Use the echo tool from the everything MCP server with message 'OpenTrad evidence sample A'."
```

- **结果**:exit 0,9 行 NDJSON,duration 13.077s,total cost $0.151996

## Redact 范围

为保护本机环境,以下字段做了脱敏(用 `jq` 处理):

- `cwd`:`/Users/<原用户名>/Desktop/...` → `/Users/<USER>/Desktop/...`
- `memory_paths`:整字段已删(本机自定义 memory 路径)
- `plugins`:整字段已删(本机自定义 Claude Code plugins 列表)

其他字段(session_id 是占位 UUID;事件 uuid / msgId / tool_use_id 是 CC 内部生成、不暴露身份;timestamps、cost、token usage 等)**保留原值**,以保留事件流的关联性供下游施工参考。

## 完整事件流(9 行 NDJSON)

下面 9 行按时间顺序展示。每行后跟字段标注,标注里的"💡"是供下游(M1 #8 / #25)施工参考的关键发现。

### 行 1:`system/init`

```ndjson
{"type":"system","subtype":"init","cwd":"/Users/<USER>/Desktop/open-trad/opentrad","session_id":"11111111-1111-1111-1111-111111111111","tools":["Task","AskUserQuestion","Bash","CronCreate","CronDelete","CronList","Edit","EnterPlanMode","EnterWorktree","ExitPlanMode","ExitWorktree","Glob","Grep","ListMcpResourcesTool","Monitor","NotebookEdit","PushNotification","Read","ReadMcpResourceTool","RemoteTrigger","ScheduleWakeup","Skill","TaskOutput","TaskStop","TodoWrite","ToolSearch","WebFetch","WebSearch","Write","mcp__everything__echo","mcp__everything__get-annotated-message","mcp__everything__get-env","mcp__everything__get-resource-links","mcp__everything__get-resource-reference","mcp__everything__get-roots-list","mcp__everything__get-structured-content","mcp__everything__get-sum","mcp__everything__get-tiny-image","mcp__everything__gzip-file-as-resource","mcp__everything__simulate-research-query","mcp__everything__toggle-simulated-logging","mcp__everything__toggle-subscriber-updates","mcp__everything__trigger-elicitation-request","mcp__everything__trigger-long-running-operation"],"mcp_servers":[{"name":"everything","status":"connected"}],"model":"claude-opus-4-7","permissionMode":"default","slash_commands":[...],"apiKeySource":"none","claude_code_version":"2.1.119","output_style":"default","agents":["Explore","general-purpose","Plan","statusline-setup"],"skills":[...],"analytics_disabled":false,"uuid":"49ad36f6-e4dd-4023-a8a4-9b8b6da46b54","fast_mode_state":"off"}
```

> 💡 **MCP tool 命名格式**:`mcp__<server-name>__<tool-name>`。这里 server name = `"everything"`(取自 mcp-config 的 `mcpServers` key),tool name 含连字符如 `get-annotated-message`、`toggle-simulated-logging`。**M1 #25 实施**:OpenTrad 自己的 server 名建议用 `"opentrad"`,工具名如 `mcp__opentrad__draft_save`(下划线分隔在工具名内部也可,见 server-everything 的 `get-env` 用连字符,Anthropic 官方建议用 snake_case 但 server-everything 用 kebab-case;**OpenTrad 选 snake_case**,与 02 §四 / 03 §4.4 一致)。
>
> 💡 **`mcp_servers` 字段**:`[{"name": "everything", "status": "connected"}]`。CC 拉起 MCP server 后会上报 connected 状态。**M1 #25 graceful degrade 时**:如果 IPC bridge 连接失败,server 仍能 stdio 启动并被 CC 识别为 connected,只是 tool 调用时会 deny(M1 #8 决策 D-M1-5)。
>
> 💡 **新字段**:wire schema 用 `.passthrough()` 兜底的字段,`analytics_disabled` / `fast_mode_state` / 删除前的 `memory_paths` / `plugins`。M0 wire schema 没识别,passthrough OK。

### 行 2:`rate_limit_event`

```ndjson
{"type":"rate_limit_event","rate_limit_info":{"status":"allowed","resetsAt":1777118400,"rateLimitType":"five_hour","overageStatus":"rejected","overageDisabledReason":"out_of_credits","isUsingOverage":false},"uuid":"86902a2b-86e0-4b1c-b9f9-06e566b6d739","session_id":"11111111-1111-1111-1111-111111111111"}
```

> 这是 task 开始前的限速心跳。M0 stream-parser 已处理(`RateLimitEventSchema`)。

### 行 3:`assistant`(thinking)

```ndjson
{"type":"assistant","message":{"model":"claude-opus-4-7","id":"msg_01AVxzjGtSkFb9Wx92bTnem6","type":"message","role":"assistant","content":[{"type":"thinking","thinking":"","signature":"Er4CClkIDRgC...AGAE="}],"stop_reason":null,"stop_sequence":null,...,"usage":{"input_tokens":6,"cache_creation_input_tokens":9029,...},"context_management":null},"parent_tool_use_id":null,"session_id":"11111111-1111-1111-1111-111111111111","uuid":"0cddebae-6c44-4dbc-a33c-8421b6f0186f"}
```

> 💡 **`thinking` block 的 `signature`**:Anthropic API 要求保留(D6b)。M0 normalize 已经把它映射到 `AssistantThinkingEvent.signature`。
>
> 💡 **每个 wire event 内只有 1 个 content block**:验证 retrospective §三 D6 后续的实测——`content.length === 1`,正是"per-wire-event 语义"的根因(详见 [open-trad/opentrad packages/stream-parser/README.md](https://github.com/open-trad/opentrad/blob/main/packages/stream-parser/README.md) Consumer guide)。
>
> 💡 **`stop_reason: null`**:本 wire event 不是整条消息的终态。CC 会发后续 wire event。

### 行 4:`assistant`(tool_use:CC 内置 ToolSearch)

```ndjson
{"type":"assistant","message":{"model":"claude-opus-4-7","id":"msg_01AVxzjGtSkFb9Wx92bTnem6",...,"content":[{"type":"tool_use","id":"toolu_01JDmwaAnTFDHe9UkUU4b84J","name":"ToolSearch","input":{"query":"select:mcp__everything__echo","max_results":1},"caller":{"type":"direct"}}],...,"uuid":"8c5b13fb-7131-4949-af78-b6fe48b2160b"}
```

> 💡 **CC 自己用了 `ToolSearch`**:CC 2.1.119 在调用 MCP tool 前会先用内置 `ToolSearch` 找到 tool schema(避免每次任务都把所有 tool schema 注入 prompt)。**M1 #25 不需要拦截 ToolSearch**(CC 内建,Risk Gate 只管 OpenTrad 自己暴露的 tools)。
>
> 💡 **新字段 `caller: {"type": "direct"}`**:CC 2.1.119 的 tool_use 携带 caller 信息(可能是为子任务/嵌套工具准备的)。M0 wire schema 没识别,passthrough 兜底。**M1 #25 暂不需要解读**。
>
> 💡 **`msg_id` 与行 3 相同**(`msg_01AVxzjGtSkFb9Wx92bTnem6`):同一逻辑 message 的连续 wire events 共享 `msg_id`,但每个 wire event 独立扁平化(D6 后续修订)。

### 行 5:`user`(tool_result:ToolSearch 返回 echo schema reference)

```ndjson
{"type":"user","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_01JDmwaAnTFDHe9UkUU4b84J","content":[{"type":"tool_reference","tool_name":"mcp__everything__echo"}]}]},"parent_tool_use_id":null,"session_id":"11111111-1111-1111-1111-111111111111","uuid":"09679311-135e-414b-abab-081aeb18e362","timestamp":"2026-04-25T08:38:24.897Z","tool_use_result":{"matches":["mcp__everything__echo"],"query":"select:mcp__everything__echo","total_deferred_tools":34}}
```

> 💡 **`tool_result` 在 `user` message 中**:验证了 `cc-event.ts` line 135 的 TODO 假设——tool_result 的 wire 源是 user message 的 content block,不是 assistant。
>
> 💡 **新 content type `tool_reference`**:ToolSearch 的特殊返回格式(`{"type": "tool_reference", "tool_name": "..."}`),只用于 ToolSearch 这一类内置工具的结果,**不**是普通 MCP tool 的返回结构。M0 wire schema 应该用 unknown content 兜底。
>
> 💡 **`timestamp` 字段**:user message 带 ISO8601 时间戳,assistant message 没有(由 LLM 生成,无明确时间点)。
>
> 💡 **`tool_use_result` 字段**:CC 在 user message 顶层重复了 tool 的结构化结果(便于消费方不展开 content[] 也能拿到)。**M1 #25 设计**:OpenTrad 的 mcp-server 返回结果时不需要构造这个字段——CC 自己加。

### 行 6:`assistant`(tool_use:实际调用 mcp__everything__echo)

```ndjson
{"type":"assistant","message":{"model":"claude-opus-4-7","id":"msg_018T4AJ84KuwMNM9qNtVFjUP",...,"content":[{"type":"tool_use","id":"toolu_014Y957Qcbz7rwwy4p2aPGzy","name":"mcp__everything__echo","input":{"message":"OpenTrad evidence sample A"},"caller":{"type":"direct"}}],...,"diagnostics":{"cache_miss_reason":{"type":"system_changed","cache_missed_input_tokens":24791}},"context_management":null},...,"uuid":"08d54a86-7994-4cae-99f2-1fdcf7a090cb"}
```

> 💡 **新 message id**(`msg_018T4AJ84KuwMNM9qNtVFjUP`,与行 3-4 不同):CC 把 ToolSearch 调用和 echo 调用拆成了两个 logical message。**对 stream-parser 没影响**(每条 wire event 独立扁平化),但对 UI 层"消息说完"的判断有意义——新 msgId 即是"前一个 message 已结束"的锚点之一(详见 stream-parser README Consumer guide 锚点 3)。
>
> 💡 **新字段 `diagnostics: {cache_miss_reason}`**:CC 2.1.119 给消费方的诊断信息(便于 cost 优化分析)。passthrough 兜底,M1 不需要解读。

### 行 7:`user`(tool_result:echo 返回 text)

```ndjson
{"type":"user","message":{"role":"user","content":[{"tool_use_id":"toolu_014Y957Qcbz7rwwy4p2aPGzy","type":"tool_result","content":[{"type":"text","text":"Echo: OpenTrad evidence sample A"}]}]},...,"timestamp":"2026-04-25T08:38:29.338Z","tool_use_result":[{"type":"text","text":"Echo: OpenTrad evidence sample A"}]}
```

> 💡 **MCP tool 的 result 结构**:`content: [{"type": "text", "text": "..."}]`。这是普通 MCP tool 调用结果的 wire 形态(对比行 5 的 ToolSearch 特殊形态 `tool_reference`)。**M1 #25 实施**:OpenTrad MCP server 的 tool 返回 `{ content: [{ type: "text", text: "..." }] }` 即可。

### 行 8:`assistant`(text:最终答复)

```ndjson
{"type":"assistant","message":{"model":"claude-opus-4-7","id":"msg_01TofaNafC4Lyu6xjQc5qKN6",...,"content":[{"type":"text","text":"Echo 返回：`Echo: OpenTrad evidence sample A`。"}],...,"diagnostics":null,...},...,"uuid":"98de74cd-691c-45f6-9636-deaf7ae22794"}
```

> 💡 **第三个 msgId**(`msg_01TofaNafC4Lyu6xjQc5qKN6`):每次 LLM 转折都开新 msgId。

### 行 9:`result`(任务结束)

```ndjson
{"type":"result","subtype":"success","is_error":false,"api_error_status":null,"duration_ms":13077,"duration_api_ms":15935,"num_turns":3,"result":"Echo 返回：`Echo: OpenTrad evidence sample A`。","stop_reason":"end_turn","session_id":"11111111-1111-1111-1111-111111111111","total_cost_usd":0.15199600000000002,"usage":{...},"modelUsage":{"claude-haiku-4-5-20251001":{"inputTokens":359,"outputTokens":16,...,"costUSD":0.000439,...},"claude-opus-4-7":{"inputTokens":13,"outputTokens":272,...,"costUSD":0.151557,...}},"permission_denials":[],"terminal_reason":"completed","fast_mode_state":"off","uuid":"2e7174bf-a1f0-4108-8fa3-c42c28ba0469"}
```

> 💡 **`permission_denials: []`**:正常路径下为空数组。disallowed 路径见 [`02-tool-disallowed-flow.md`](./02-tool-disallowed-flow.md)。
>
> 💡 **`modelUsage` 含两个模型**:`claude-haiku-4-5-20251001`(CC 内部 ToolSearch / 路由用 haiku 节省 cost)+ `claude-opus-4-7`(主对话)。**M1 #28 RiskGate audit log 字段设计**:可能要分别记录 cost。
>
> 💡 **`terminal_reason: "completed"`**:正常结束。其他可能值需后续抓样本(超时 / 取消 / api_error)。

## 关键发现汇总(给 M1 #8 / #25 / #28)

| 发现 | 影响哪个 issue |
|---|---|
| MCP tool 命名格式 `mcp__<server>__<tool>` | M1 #8(server name 选 `opentrad`)/ M1 #28(skill manifest `allowedTools` 写法) |
| CC 用内置 `ToolSearch` 加载 tool schema(每次任务) | M1 #8 不需要拦截 ToolSearch |
| `tool_result` 出现在 user message 的 content[] 里 | confirms `cc-event.ts` line 135 TODO 假设 |
| `tool_reference` 是 ToolSearch 特殊 result type | wire schema 用 unknown content 兜底足够 |
| `caller`、`diagnostics`、`memory_paths` 等新字段 | passthrough 兜底,M1 暂不解读 |
| 每个 wire event content[] 长度 = 1 | 印证 D6 后续 per-wire-event 语义(已在 architecture §4.2 勘误 3 + stream-parser README 落地) |
| `permission_denials` 是 result event 的 wire 字段 | M1 #28 deny 路径参考(见 sample b) |
| `modelUsage` 含多模型 cost 拆分 | M1 #28 audit log + 计费设计参考 |
