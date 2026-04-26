# Evidence sample 03:RiskGate 三场景 wire & audit trace

**采集日期**:2026-04-26
**Claude Code 版本**:2.1.119
**关联 issue**:M1 #11 / [open-trad/opentrad#28](https://github.com/open-trad/opentrad/issues/28)
**关联 PR**:M1 #28 实施 PR(待开)
**性质**:**基于代码推导的 expected wire & audit trace**(非真机采集)— dev e2e 时发起人需校实际行为是否一致。代码层面已 typecheck / lint / unit test 全绿。

## 用途

记录 RiskGate 三种典型决策路径的 wire trace + audit_log 行。供 M1 retrospective 时:
1. 验证"双向 IPC 设计"是否如设计执行
2. 校 graceful degrade 4 重护栏(详见 M1 #28 PR description 链路图)
3. 与 #1 evidence/m1-prep-samples/02 的 CC 内建 disallowed 路径对照(本文是 OpenTrad RiskGate 拦截路径)

## 三场景

| # | 场景 | RiskGate 4 步路径 | automated |
|---|---|---|---|
| A | blocked 工具直接 deny | 步骤 1 命中 | true |
| B | review 工具 → 用户拒绝 | 步骤 4 promptUser → user deny | false |
| C | allow_always 第二次调用 → automated allow | 步骤 3 rule_matched | true |

## Redact 范围

- `sessionId` → `<REDACTED-SESSION-A>` / `-B` / `-C`(每场景独立)
- `requestId` → `<REDACTED-REQID-N>`
- `timestamp` → `<TS-N>`(epoch ms)
- 其他参数(URL / 邮箱 / 路径)按 `redact.ts` 规则脱敏(`a***@x.com` / `/<HOME>/<REDACTED>` 等)

## 双向 IPC bridge 协议(本文涉及的帧)

mcp-server ↔ desktop 通过 Unix domain socket(macOS/Linux)/ named pipe(Windows)+ NDJSON,JSON-RPC 2.0。每场景的 wire trace 段都展示 mcp-server 端发出 / desktop 端发出的帧。

renderer ↔ main 通过 IPC channel `risk-gate:confirm`(main → renderer push)+ `risk-gate:response`(renderer → main invoke)。**与 IPC bridge wire 解耦**(发起人 #25 hello 帧约束)。

---

## 场景 A:blocked 工具直接 deny(步骤 1)

### 触发

```
skill: trade-email-writer (假设含 mcp__opentrad__execute_shell, riskLevel='blocked')
user inputs: { topic: "test" }
CC 决定调用:mcp__opentrad__execute_shell({ cmd: "rm -rf /" })
```

> 注:M1 fixture-skill 实际无 blocked 工具,本场景假设有一个 hypothetical 的 blocked 工具。M2 加真实 blocked 工具(例:不可逆破坏性 shell 命令)时本样本为参考。

### IPC bridge wire trace

**mcp-server → desktop**(`risk-gate.request`):

```ndjson
{"jsonrpc":"2.0","id":1,"method":"risk-gate.request","params":{"skillId":"","toolName":"execute_shell","riskLevel":"blocked","params":{"cmd":"rm -rf /"}}}
```

> `skillId:""` 是 mcp-server 端占位,desktop 端用 `ctx.sessionId`(来自 hello 帧)重查真 skillId(详见 PR description "wire 协议 0 改动"段)。

**desktop → mcp-server**(`risk-gate.request` response):

```ndjson
{"jsonrpc":"2.0","id":1,"result":{"decision":"deny","reason":"blocked_policy","timestamp":<TS-A>}}
```

### renderer ↔ main 内部 channel

**无 channel 流量**:RiskGate 步骤 1 直接 deny,不调 promptUser,renderer 不弹窗。

### audit_log 写入

```sql
INSERT INTO audit_log VALUES (
  id          = auto,
  timestamp   = <TS-A>,
  session_id  = '<REDACTED-SESSION-A>',
  skill_id    = 'trade-email-writer',
  tool_name   = 'execute_shell',
  business_action = NULL,
  params_json = '{"cmd":"rm -rf /"}',
  decision    = 'deny',
  automated   = 1,
  reason      = 'blocked_policy'
);
```

### CC(MCP)收到

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "tool execute_shell is blocked by risk policy" }
  ]
}
```

CC 收到后会自己决定下一步(M1 不强制中止;通常 CC 在 assistant_text 解释错误后任务继续或 result error)。

---

## 场景 B:review 工具 → 用户拒绝(步骤 4)

### 触发

```
skill: trade-email-writer (含 browser_open, riskLevel='review')
user inputs: { topic: "RFQ for X" }
CC 决定调用:mcp__opentrad__browser_open({ url: "https://1688.com/seller/abc?contact=alice@example.com" })
```

### IPC bridge wire trace

**mcp-server → desktop**(`risk-gate.request`):

```ndjson
{"jsonrpc":"2.0","id":2,"method":"risk-gate.request","params":{"skillId":"","toolName":"browser_open","riskLevel":"review","params":{"url":"https://1688.com/seller/abc?contact=alice@example.com"}}}
```

### renderer ↔ main 内部 channel

**main → renderer**(`risk-gate:confirm`,IpcChannels.RiskGateConfirm 推送):

```json
{
  "requestId": "<REDACTED-REQID-B>",
  "sessionId": "<REDACTED-SESSION-B>",
  "skillId": "trade-email-writer",
  "toolName": "browser_open",
  "riskLevel": "review",
  "params": { "url": "https://1688.com/seller/abc?contact=alice@example.com" },
  "businessAction": null,
  "category": "browser"
}
```

> `businessAction: null` → renderer 渲染 `RiskGateDialog`(工具级);若非空 → `BusinessActionCard`(业务级)。

UI 展示参数时调 `paramsToDisplayString`(`redact.ts`)脱敏:`alice@example.com` → `a***@example.com`。

**用户操作**:点击"拒绝"按钮(decided after ~3s)。

**renderer → main**(`risk-gate:response`,IpcChannels.RiskGateResponse invoke):

```json
{
  "requestId": "<REDACTED-REQID-B>",
  "kind": "deny"
}
```

(`reason` 字段可空;用户主动 deny 时 main 端写 `audit.reason = null`)

### desktop main 内部

`prompter.resolveDecision(<REDACTED-REQID-B>, "deny", undefined)` →
`RiskGate.check` 内 promptUser promise resolve → 步骤 4 mappedDecision='deny' →
`audit.append(...)` → 返回 `{ decision: 'deny', reason: undefined, automated: false }`。

### IPC bridge response

**desktop → mcp-server**:

```ndjson
{"jsonrpc":"2.0","id":2,"result":{"decision":"deny","timestamp":<TS-B-RESPONSE>}}
```

### audit_log 写入

```sql
INSERT INTO audit_log VALUES (
  id              = auto,
  timestamp       = <TS-B>,
  session_id      = '<REDACTED-SESSION-B>',
  skill_id        = 'trade-email-writer',
  tool_name       = 'browser_open',
  business_action = NULL,
  params_json     = '{"url":"https://1688.com/seller/abc?contact=alice@example.com"}',  -- 后端原值,UI 才脱敏
  decision        = 'deny',
  automated       = 0,
  reason          = NULL
);
```

### CC(MCP)收到

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "user denied browser_open" }
  ]
}
```

CC 通常会选择 `assistant_text` 解释拒绝、再尝试不同 url 或建议用户手动操作。事件流不中断。

---

## 场景 C:allow_always 第二次调用 → automated allow(步骤 3 rule_matched)

### 触发(两次调用对比)

#### 第一次调用(同场景 B 但用户选"以后都允许")

renderer 用户点击 **以后都允许**:

```json
{ "requestId": "<REDACTED-REQID-C-1>", "kind": "allow_always" }
```

`RiskGate.check` 走步骤 4 → `mappedDecision='allow_always'` → 调 `RuleProvider.save` 写 risk_rules:

```sql
-- M1 #28 RuleProvider.save 写规则:
INSERT INTO risk_rules VALUES (
  id              = auto,
  skill_id        = 'trade-email-writer',
  tool_name       = 'browser_open',  -- 工具级:business_action=NULL
  business_action = NULL,
  decision        = 'allow',
  created_at      = <TS-C-1>
);
-- audit_log 同时写一条 user 决策:
INSERT INTO audit_log VALUES (
  ...
  decision = 'allow_always',
  automated = 0,
  reason = NULL
);
```

#### 第二次调用(同 skillId + toolName,任意 url)

新会话 / 同会话 `<REDACTED-SESSION-C>`,CC 调用:

```json
{ "url": "https://1688.com/another-seller" }
```

**mcp-server → desktop**(`risk-gate.request`):

```ndjson
{"jsonrpc":"2.0","id":3,"method":"risk-gate.request","params":{"skillId":"","toolName":"browser_open","riskLevel":"review","params":{"url":"https://1688.com/another-seller"}}}
```

`RiskGate.check`:
- 步骤 1 not blocked
- 步骤 2 not (safe + 无 businessAction)(review)
- **步骤 3 rule_matched**:`DbRuleProvider.findMatching({skillId:'trade-email-writer', toolName:'browser_open', businessAction:null})` 查到第一次写的规则 → `decision='allow_always', automated=true, reason='rule_matched'`
- **不调 promptUser**(renderer 不弹窗,UX 静默)

### audit_log 写入(对比第一次)

```sql
-- 第一次(场景 C-1, user_decision):
... decision='allow_always', automated=0, reason=NULL ...

-- 第二次(本次,rule_matched 自动):
INSERT INTO audit_log VALUES (
  id              = auto,
  timestamp       = <TS-C-2>,
  session_id      = '<REDACTED-SESSION-C>',
  skill_id        = 'trade-email-writer',
  tool_name       = 'browser_open',
  business_action = NULL,
  params_json     = '{"url":"https://1688.com/another-seller"}',
  decision        = 'allow_always',
  automated       = 1,        -- 关键差别:automated=1
  reason          = 'rule_matched'  -- 关键差别:reason 标 rule_matched
);
```

**对比关键点**(audit_log 区分用户决策 vs 规则命中):

| 字段 | 第一次 user allow_always | 第二次 rule_matched |
|---|---|---|
| `decision` | `allow_always` | `allow_always`(同) |
| `automated` | `0` | `1` |
| `reason` | `NULL` | `rule_matched` |

### IPC bridge response

**desktop → mcp-server**:

```ndjson
{"jsonrpc":"2.0","id":3,"result":{"decision":"allow_always","reason":"rule_matched","timestamp":<TS-C-2>}}
```

middleware 判 `decision === "allow_always"` → `{allowed: true}` → tool execute 继续,无延迟。

### CC(MCP)收到

正常 tool result,无 error 字段:

```json
{
  "content": [
    { "type": "text", "text": "{ \"pageId\": \"...\", \"title\": \"...\", \"url\": \"https://1688.com/another-seller\" }" }
  ]
}
```

---

## 验证步骤(发起人 dev e2e)

发起人在 mac dev 模式跑 trade-email-writer skill 时:

1. **场景 A**:M1 fixture-skill 无 blocked 工具,跳过(M2 真做)
2. **场景 B**:点击 SkillPicker fixture-skill → 输入 → 触发 browser_open(需要 fixture skill 加 browser_open 到 allowedTools;M1 实测时可临时改 fixture)→ RiskGateDialog 弹出 → 点拒绝 → CC 收到 deny
   ```bash
   # 观察 audit_log:
   sqlite3 ~/.opentrad/opentrad.db "SELECT * FROM audit_log ORDER BY timestamp DESC LIMIT 5;"
   ```
3. **场景 C**:同场景 B,但点"以后都允许"。然后再次同样调用 → 应**不弹窗**,直接 allow。
   ```bash
   # 观察 risk_rules + audit_log 对比:
   sqlite3 ~/.opentrad/opentrad.db "SELECT * FROM risk_rules;"
   sqlite3 ~/.opentrad/opentrad.db "SELECT decision, automated, reason FROM audit_log ORDER BY timestamp DESC LIMIT 5;"
   ```

观察到的真实 audit_log tail 提交到 #28 PR comment(通报点 3)。如形态与本 evidence 一致 → ✓ M1 #28 验收。

---

## 与 #1 evidence/02-tool-disallowed-flow.md 对比

| 维度 | #1 sample 02(CC 内建 disallowed) | 本 sample 03(OpenTrad RiskGate deny) |
|---|---|---|
| 拦截位置 | CC 客户端层(allowedTools 白名单) | mcp-server middleware → IPC bridge → desktop RiskGate |
| CC 收到形态 | `is_error:true` + string content | 结构化 MCP error(`isError:true` + content array) |
| Wire 表达 | result 事件含 `permission_denials` 数组 | tool_result 事件含 isError(同 CC 任意 error) |
| audit_log | 不写(CC 内建拦截 OpenTrad 不感知) | **每次都写**(automated allow / deny / user 全记录) |
| 用户决策 | 无(白名单是静态配置) | **4 种**:allow_once / allow_always / deny / request_edit |
| 5min 超时 | 无 | **有**(main 进程 UserPrompter,A6) |

两条路径互补:CC 内建 allowedTools 白名单是 first-line 静态过滤(M1 设 `--allowedTools` 时就锁住);OpenTrad RiskGate 是 second-line 动态拦截(运行时用户介入)。

## 不在本 sample 内 / 后续

- 真机 wire trace 采集(本 sample 是 expected,M1 #28 真合后 mac dev 校验时替换)
- 业务级触发样本(stopBeforeList 命中 → BusinessActionCard):M1 fixture-skill 无 stopBefore;M1 #30 真 trade-email-writer 加 `stopBefore: [send_email]` 后补
- request_edit 决策的实际 CC 行为(D-M1-6 v1 等价 deny + reason='user_requested_edit',CC 是否真重生成 input?dev e2e 校)
