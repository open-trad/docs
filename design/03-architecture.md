# OpenTrad v1 架构设计

**文档版本**：1.0  
**撰写日期**：2026-04-24  
**状态**：已审阅 
**作者**：总工（Claude Opus 4.7）  
**依赖**：`01-positioning.md` v1.1、`02-mvp-slicing.md` v1.0

---

## 本文档的用途

这份文档是给施工单位（Claude Code）开工前的**技术蓝图**。回答三类问题：

1. **怎么组织代码**：目录结构、包划分、技术栈
2. **关键接口长什么样**：模块间 API、数据结构、事件协议
3. **为什么这么选**：技术选型的决策记录（ADR），防止后来者反复折腾

读完这份文档，Claude Code 应该能直接开始写第一行代码，不需要再来问"用什么框架、文件放哪、接口怎么定"。

---

## 一、技术栈总览

| 层次 | 选型 | 替代选项 | 选它的理由（详见 ADR） |
|------|------|---------|---------------------|
| 桌面框架 | **Electron 34+** | Tauri 2 | 团队 TS/Node 熟悉度高，node-pty/xterm.js 生态成熟，发布链路完善 |
| UI 框架 | **React 18 + TypeScript 5** | Vue / Svelte | 生态最大，Claude Code 施工时样板代码最多 |
| UI 组件库 | **shadcn/ui + Tailwind CSS 4** | Ant Design / MUI | 不是"一套组件"是"可复制的源码"，符合开源项目气质 |
| 状态管理 | **Zustand 5** | Redux / Jotai | 轻量、无模板代码、对 TS 友好 |
| 路由 | **React Router 7** | TanStack Router | 项目不复杂，用最主流的 |
| 数据持久化 | **better-sqlite3** | LevelDB / JSON 文件 | 同步 API 省事、抄 Accio 的成熟方案 |
| 进程通信 | **node-pty + child_process** | deno subprocess | 包装 CC CLI 的标准方案 |
| Terminal 渲染 | **xterm.js 5** | 无替代 | 事实标准 |
| Markdown | **react-markdown + remark-gfm** | markdown-it | React 原生集成 |
| 浏览器控制 | **Playwright 1.5+** | Puppeteer / CDP 直连 | 比 Puppeteer 现代，比 CDP 直连 API 友好 |
| MCP SDK | **@modelcontextprotocol/sdk** | 自研 | MCP 官方 TS SDK |
| 打包 | **electron-builder** | electron-forge | 跨平台签名和自动更新链路成熟 |
| 包管理器 | **pnpm 10** | npm / yarn | monorepo 友好、磁盘占用少 |
| 构建工具 | **Vite 6 + electron-vite** | webpack | 开发热重载快 |
| i18n | **i18next + react-i18next** | lingui | 社区规模最大 |
| 代码质量 | **Biome 2** | ESLint + Prettier | 一个工具替代两个，速度快 10 倍 |
| 测试 | **Vitest + Playwright** | Jest | Vite 原生集成、同一个配置 |
| CI/CD | **GitHub Actions** | 无替代 | repo 在 GitHub 上，零配置 |

技术选型的关键决策会在**第八章 ADR** 里详细论证。

---

## 二、monorepo 结构

OpenTrad 是一个 **monorepo**，用 pnpm workspaces 管理。理由：`app` / `mcp-server` / `skills` / `shared` 等多个包会共享类型定义和工具函数，monorepo 比多 repo 省心。

### 顶层结构

```
opentrad/                       # repo 根
├── apps/
│   ├── desktop/                # Electron 桌面应用（主交付物）
│   └── mcp-server/             # OpenTrad 自己的 MCP server（被 CC 调用）
├── packages/
│   ├── shared/                 # 共享类型、常量、utils
│   ├── stream-parser/          # CC stream-json 解析器（独立包，可单测）
│   ├── skill-runtime/          # Skill manifest 加载 + prompt 组装
│   ├── risk-gate/              # Risk Gate 策略引擎
│   ├── cc-adapter/             # Claude Code 进程管理和版本适配
│   └── browser-tools/          # Playwright 封装的浏览器工具（给 mcp-server 用）
├── skills/                     # v1 内置的 5 个 P0 skill（markdown+yaml）
│   ├── product-sourcing-1688/
│   ├── supplier-rfq-draft/
│   ├── trade-email-writer/
│   ├── listing-localizer-seo/
│   └── hs-code-compliance-check/
├── docs/                       # 设计文档（本文档系列放这里）
│   ├── 01-positioning.md
│   ├── 02-mvp-slicing.md
│   ├── 03-architecture.md      # 本文档
│   └── adr/                    # 单独的 ADR 记录
├── .github/
│   ├── workflows/              # CI/CD
│   └── ISSUE_TEMPLATE/
├── scripts/                    # 开发辅助脚本
├── package.json                # 根 package.json
├── pnpm-workspace.yaml
├── biome.json
├── tsconfig.base.json
├── .gitignore
├── LICENSE                     # AGPL-3.0
├── README.md
└── CONTRIBUTING.md
```

### `apps/desktop` 内部结构

```
apps/desktop/
├── src/
│   ├── main/                   # Electron 主进程（Node.js）
│   │   ├── index.ts            # 入口
│   │   ├── window.ts           # BrowserWindow 管理
│   │   ├── ipc/                # IPC handlers
│   │   │   ├── cc.ts           # CC 相关 IPC
│   │   │   ├── skill.ts        # Skill 相关 IPC
│   │   │   ├── session.ts      # Session 相关 IPC
│   │   │   └── settings.ts
│   │   ├── services/
│   │   │   ├── cc-manager.ts   # Process Manager 的主进程实现
│   │   │   ├── mcp-writer.ts   # MCP Config Writer
│   │   │   ├── session-store.ts # SQLite session 持久化
│   │   │   ├── installer.ts    # CC 安装检测和引导
│   │   │   └── audit-log.ts    # Risk Gate 审计日志
│   │   └── preload.ts          # 预加载脚本
│   ├── renderer/               # React 渲染层
│   │   ├── index.tsx           # 入口
│   │   ├── App.tsx
│   │   ├── layout/
│   │   │   ├── MainLayout.tsx  # 三栏布局
│   │   │   ├── Sidebar.tsx     # 左侧栏
│   │   │   ├── Chat.tsx        # 主工作区
│   │   │   └── RightPanel.tsx  # 右侧辅助栏
│   │   ├── components/         # 通用组件
│   │   │   ├── ui/             # shadcn 组件
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── ToolCallCard.tsx
│   │   │   ├── RiskGateDialog.tsx
│   │   │   ├── SkillPicker.tsx
│   │   │   └── InstallWizard.tsx
│   │   ├── features/           # 功能模块
│   │   │   ├── onboarding/
│   │   │   ├── chat/
│   │   │   ├── skills/
│   │   │   ├── history/
│   │   │   └── settings/
│   │   ├── stores/             # Zustand stores
│   │   │   ├── session.ts
│   │   │   ├── skill.ts
│   │   │   ├── settings.ts
│   │   │   └── ui.ts
│   │   ├── hooks/
│   │   ├── lib/                # utils
│   │   └── i18n/
│   │       ├── zh-CN.json
│   │       └── en.json
│   └── common/                 # 主进程和渲染层共享（类型、常量）
├── electron.vite.config.ts
├── electron-builder.yml
├── package.json
└── tsconfig.json
```

### `apps/mcp-server` 内部结构

```
apps/mcp-server/
├── src/
│   ├── index.ts                # MCP server 入口（stdio transport）
│   ├── tools/                  # 所有暴露给 CC 的工具
│   │   ├── browser/
│   │   │   ├── open.ts
│   │   │   ├── read.ts
│   │   │   └── screenshot.ts
│   │   ├── platforms/
│   │   │   ├── 1688-search.ts
│   │   │   ├── alibaba-supplier.ts
│   │   │   └── hs-code-lookup.ts
│   │   ├── drafts/
│   │   │   └── save.ts
│   │   └── index.ts
│   ├── middleware/
│   │   └── risk-gate.ts        # 工具调用前的 Risk Gate 拦截
│   ├── ipc-bridge.ts           # 和 desktop 主进程的 IPC 桥（Risk Gate 弹窗需要）
│   └── config.ts
├── package.json
└── tsconfig.json
```

**关键设计**：MCP server 作为独立进程运行（CC 通过 stdio 拉起），但通过 **IPC 桥**（Unix socket 或 named pipe）和 desktop 主进程通信，这样 Risk Gate 弹窗可以在 desktop UI 里出现。

---

## 三、运行时架构

### 进程拓扑

应用运行时有 **4 类进程**：

```
┌────────────────────────────────────────────────────────────┐
│ 1. Electron Main Process (Node.js)                         │
│    - 窗口管理                                               │
│    - CC 子进程管理（spawn / kill / 通信）                    │
│    - SQLite 访问                                            │
│    - IPC bridge server                                      │
└────────────────────────────────────────────────────────────┘
        ▲                  ▲                      ▲
        │ IPC              │ stdout/stdin         │ IPC bridge
        ▼                  ▼                      ▼
┌──────────────┐  ┌──────────────────┐  ┌─────────────────┐
│ 2. Renderer  │  │ 3. Claude Code   │  │ 4. OpenTrad MCP │
│    Process   │  │    CLI           │  │    Server       │
│  (Chromium)  │  │  (native binary) │  │   (Node.js)     │
│              │  │                  │  │                 │
│  React UI    │  │  - LLM 调用      │  │  - Tools        │
│              │  │  - Tool routing  │  │  - Risk Gate    │
│              │  │  - Skill prompt  │  │  - Playwright   │
└──────────────┘  └──────────────────┘  └─────────────────┘
                          ▲                      │
                          │  stdio MCP           │
                          └──────────────────────┘
```

### 一次任务的完整数据流

以"用户跑 `trade-email-writer` skill 写一封报价邮件"为例：

```
1. 用户在 UI 填表 + 点"发送"
   │
   ├─ [Renderer] 组合 skill inputs + prompt template
   │
   └─ [Renderer → Main via IPC] 请求启动 CC 任务
       │
       └─ [Main] CCManager.spawn()
           ├─ 生成 session UUID
           ├─ 写 opentrad-<session>.mcp.json（临时文件）
           ├─ spawn claude -p --session-id X --mcp-config Y --output-format stream-json
           └─ 同时启动 OpenTrad MCP server（CC 通过 stdio 拉起）

2. CC 读到 system prompt + user input
   │
   ├─ 决定调用工具 (比如 mcp__opentrad__browser_open)
   │
   └─ [CC → MCP server via stdio] tool call
       │
       ├─ [MCP server] RiskGate 判断：risk_level=safe？
       │   ├─ safe  → 直接执行
       │   ├─ review → [MCP server → Main via IPC bridge] 请求用户确认
       │   │           [Main → Renderer via IPC] 弹确认对话框
       │   │           用户确认 → 返回到 MCP server → 执行
       │   └─ blocked → 直接返回错误给 CC
       │
       └─ 执行工具（比如 Playwright 打开 1688 搜索）
           │
           └─ 返回结果给 CC

3. CC 生成 assistant 消息
   │
   └─ [CC → Main via stdout] NDJSON 事件
       │
       └─ [Main] StreamParser 解析
           │
           └─ [Main → Renderer via IPC] 推送事件
               │
               └─ [Renderer] 更新 UI（新消息气泡、工具调用卡片）

4. 任务结束
   │
   ├─ [CC] result 事件
   │
   └─ [Main] 清理
       ├─ kill MCP server（如果需要）
       ├─ 删除临时 mcp-config 文件
       ├─ 写入 session metadata 到 SQLite
       └─ 通知 Renderer 任务完成
```

### IPC 协议设计

**Main ↔ Renderer IPC**（Electron 标准）：

使用 `contextBridge` + typed IPC，不用 `remote` 模块（已废弃）。

所有 IPC channel 名用 `<domain>:<action>` 格式：

| Channel | 方向 | 用途 |
|---------|------|------|
| `cc:start-task` | Renderer → Main | 启动新任务 |
| `cc:cancel-task` | Renderer → Main | 取消任务 |
| `cc:event` | Main → Renderer | CC stream-json 事件推送 |
| `cc:status` | 双向 | CC 状态查询/推送 |
| `skill:list` | Renderer → Main | 获取 skill 列表 |
| `skill:install` | Renderer → Main | 安装 skill |
| `session:list` | Renderer → Main | 历史 session 列表 |
| `session:resume` | Renderer → Main | 继续历史 session |
| `risk-gate:confirm` | Main → Renderer | 请求用户确认 |
| `risk-gate:response` | Renderer → Main | 用户确认结果 |
| `settings:get` / `settings:set` | 双向 | 设置读写 |

**Main ↔ MCP server IPC**（本地 socket）：

使用 Unix domain socket（macOS/Linux）或 named pipe（Windows）。协议为 JSON-RPC 2.0。主要方法：

| Method | 方向 | 用途 |
|--------|------|------|
| `risk-gate.request` | MCP → Main | 请求用户确认（等待响应） |
| `audit.log` | MCP → Main | 写审计日志 |
| `draft.save` | MCP → Main | 保存草稿到用户目录 |
| `session.metadata` | MCP → Main | 获取当前 session 元数据 |

Socket 路径：
- macOS/Linux：`~/.opentrad/ipc.sock`
- Windows：`\\.\pipe\opentrad-ipc`

**为什么不直接用 Electron IPC 跨进程？** Electron 的 IPC 只在主进程和渲染进程之间工作，MCP server 是独立 Node 进程，必须用自己的 socket。

---

## 四、模块详细设计

### 4.1 Process Manager（`packages/cc-adapter`）

核心类 `CCManager`：

```typescript
// packages/cc-adapter/src/cc-manager.ts

interface CCTaskOptions {
  sessionId: string;          // UUID
  prompt: string;             // 完整的 user prompt
  mcpConfigPath: string;      // 生成的 mcp-config 文件路径
  allowedTools: string[];     // 工具白名单
  cwd?: string;               // 工作目录
  model?: 'default' | 'haiku' | 'sonnet' | 'opus';
  permissionMode?: 'default' | 'acceptEdits' | 'bypassPermissions';
  resume?: boolean;           // 是否是续聊
}

interface CCTaskHandle {
  sessionId: string;
  pid: number;
  events: AsyncIterable<CCEvent>;   // stream-json 事件流
  cancel(): Promise<void>;
  result(): Promise<CCResult>;      // 等待最终 result
}

class CCManager {
  // 检测 CC 是否已安装
  async detectInstallation(): Promise<{ installed: boolean; version?: string; path?: string }>;

  // 检测登录状态
  async getAuthStatus(): Promise<{ loggedIn: boolean; method?: 'subscription' | 'api_key'; email?: string }>;

  // 启动任务
  async startTask(opts: CCTaskOptions): Promise<CCTaskHandle>;

  // 清理所有任务（应用退出时调用）
  async cleanup(): Promise<void>;

  // 当前所有活动任务
  get activeTasks(): ReadonlyMap<string, CCTaskHandle>;
}
```

关键实现细节：

- `startTask` 内部用 `child_process.spawn('claude', [...args])`
- stdout 是 line-buffered NDJSON，用 `readline.createInterface` 逐行读
- 子进程超时策略：默认 10 分钟无输出自动 kill（防止 CC 卡死）
- 子进程退出时触发 cleanup：kill 对应的 MCP server、删除临时文件

### 4.2 Stream Parser（`packages/stream-parser`）

核心职责：把 NDJSON 字符串流转换成 typed `CCEvent`。

```typescript
// packages/stream-parser/src/types.ts

// CC stream-json 事件的 discriminated union
type CCEvent =
  | { type: 'system'; subtype: 'init'; data: SystemInitData }
  | { type: 'assistant'; content: AssistantContent }
  | { type: 'tool_use'; toolUseId: string; name: string; input: unknown }
  | { type: 'tool_result'; toolUseId: string; content: unknown; isError?: boolean }
  | { type: 'result'; subtype: 'success' | 'error'; data: ResultData }
  | { type: 'rate_limit_event'; rateLimitInfo: RateLimitInfo }
  | { type: 'unknown'; raw: unknown };  // 未知事件作为 unknown 保留，不丢弃

interface SystemInitData {
  sessionId: string;
  tools: string[];
  mcpServers: McpServerInfo[];
  model: string;
  permissionMode: string;
  claudeCodeVersion: string;
  apiKeySource: 'subscription' | 'api_key' | 'bedrock' | 'vertex';
}

type AssistantContent =
  | { type: 'text'; text: string }
  | { type: 'thinking'; thinking: string };

// packages/stream-parser/src/parser.ts
class StreamParser {
  private buffer = '';

  *parseChunk(chunk: string): Generator<CCEvent> {
    this.buffer += chunk;
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() ?? '';  // 最后一行可能不完整，留着下次

    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const raw = JSON.parse(line);
        yield this.normalize(raw);
      } catch (err) {
        yield { type: 'unknown', raw: line };
      }
    }
  }

  private normalize(raw: any): CCEvent {
    // 根据 raw.type 分发到具体 normalizer
    // 版本兼容处理：2.1.x 和 2.2.x 的字段差异在这里吸收
  }
}
```

**Schema version guard**：
```typescript
// packages/stream-parser/src/compat.ts
const KNOWN_VERSIONS = ['2.1.0', '2.1.1', ..., '2.1.119'];
const COMPAT_MATRIX = {
  '2.1': { supported: true },
  '2.2': { supported: 'experimental' },
};

function checkCompatibility(version: string): CompatStatus {
  const minor = version.split('.').slice(0, 2).join('.');
  return COMPAT_MATRIX[minor] ?? { supported: false };
}
```

### 4.3 Skill Runtime（`packages/skill-runtime`）

```typescript
// packages/skill-runtime/src/types.ts

interface SkillManifest {
  id: string;
  title: string;
  version: string;
  description: string;
  category: 'sourcing' | 'communication' | 'listing' | 'compliance' | 'other';
  riskLevel: 'draft_only' | 'read_only' | 'interactive';
  allowedTools: string[];
  disallowedTools?: string[];
  stopBefore?: string[];      // 业务级停止动作
  inputs: SkillInput[];
  outputs: string[];
  promptTemplate: string;     // 相对 skill 目录的 markdown 文件路径
}

interface SkillInput {
  name: string;
  type: 'text' | 'textarea' | 'select' | 'url' | 'file';
  label: string;
  placeholder?: string;
  required?: boolean;
  options?: string[];         // type=select 时
  default?: string;
}

// packages/skill-runtime/src/loader.ts
class SkillLoader {
  async loadFromDirectory(dir: string): Promise<SkillManifest>;
  async loadBuiltinSkills(): Promise<SkillManifest[]>;
  async loadUserSkills(userSkillsDir: string): Promise<SkillManifest[]>;
}

// packages/skill-runtime/src/composer.ts
class PromptComposer {
  compose(skill: SkillManifest, inputs: Record<string, any>): string {
    // 1. 读取 promptTemplate 文件
    // 2. 用 mustache-style {{varName}} 替换
    // 3. 注入通用前缀（"You are operating within OpenTrad's trade-email-writer skill..."）
    // 4. 返回最终 prompt
  }
}
```

**Skill 目录结构约定**：

```
skills/trade-email-writer/
├── skill.yml           # manifest
├── prompt.md           # prompt template
├── README.md           # 用户可见说明
├── icon.svg            # skill 图标
└── examples/           # 示例（可选）
    ├── quotation.md
    └── chasing-payment.md
```

### 4.4 MCP Config Writer + OpenTrad MCP Server

**Config Writer**（`apps/desktop/src/main/services/mcp-writer.ts`）：

```typescript
class McpConfigWriter {
  async generateForSession(sessionId: string, skill: SkillManifest): Promise<string> {
    const configPath = path.join(tmpdir(), `opentrad-${sessionId}.mcp.json`);
    const config = {
      mcpServers: {
        opentrad: {
          command: process.execPath,
          args: [mcpServerEntry, '--session-id', sessionId],
          env: {
            OPENTRAD_IPC_SOCKET: ipcSocketPath,
            OPENTRAD_SESSION_ID: sessionId,
          },
        },
      },
    };
    await fs.writeFile(configPath, JSON.stringify(config, null, 2));
    return configPath;
  }
}
```

**MCP Server** 的工具注册模式：

```typescript
// apps/mcp-server/src/tools/index.ts

import { Tool } from '@modelcontextprotocol/sdk/server';

interface OpenTradTool extends Tool {
  riskLevel: 'safe' | 'review' | 'blocked';
  category: 'browser' | 'platform' | 'drafts' | 'utility';
}

export const tools: Record<string, OpenTradTool> = {
  browser_open: {
    name: 'browser_open',
    description: 'Open a URL in the managed browser',
    inputSchema: z.object({ url: z.string().url() }),
    riskLevel: 'safe',
    category: 'browser',
    async execute({ url }, ctx) {
      // 执行前经过 RiskGate middleware
      return await browserService.open(url);
    },
  },
  '1688_search': { /* ... */ },
  // ...
};
```

### 4.5 Risk Gate（`packages/risk-gate`）

```typescript
// packages/risk-gate/src/gate.ts

interface RiskGateRequest {
  skillId: string;
  toolName: string;
  params: unknown;
  riskLevel: 'safe' | 'review' | 'blocked';
  businessAction?: string;  // skill manifest 的 stop_before 字段匹配
}

interface RiskGateDecision {
  decision: 'allow' | 'deny' | 'allow_once' | 'allow_always';
  reason?: string;
  timestamp: number;
}

class RiskGate {
  async check(req: RiskGateRequest): Promise<RiskGateDecision> {
    // 1. 硬规则：blocked → 直接 deny
    if (req.riskLevel === 'blocked') {
      return { decision: 'deny', reason: 'Tool is blocked in v1', timestamp: Date.now() };
    }

    // 2. safe + 无 businessAction → 直接 allow
    if (req.riskLevel === 'safe' && !req.businessAction) {
      return { decision: 'allow', timestamp: Date.now() };
    }

    // 3. 检查用户已有规则（"以后都允许 trade-email-writer 的 browser_open"）
    const rule = await this.findMatchingRule(req);
    if (rule) return { decision: rule.decision, timestamp: Date.now() };

    // 4. 弹窗请求用户确认（通过 IPC bridge 到 Renderer）
    const response = await this.promptUser(req);
    if (response.decision === 'allow_always') {
      await this.saveRule(req, response);
    }
    await this.auditLog(req, response);
    return response;
  }
}
```

审计日志格式（JSONL，抄 Accio）：

```json
{"timestamp":"2026-04-24T10:30:00Z","sessionId":"abc","skillId":"supplier-rfq-draft","toolName":"browser_open","businessAction":null,"params":{"url":"https://1688.com/..."},"decision":"allow","automated":true}
{"timestamp":"2026-04-24T10:32:15Z","sessionId":"abc","skillId":"supplier-rfq-draft","toolName":"send_rfq","businessAction":"send_rfq","params":{"to":"<REDACTED>","subject":"RFQ for..."},"decision":"allow_once","automated":false,"userId":null}
```

---

## 五、数据持久化

### SQLite 数据库

单个 SQLite 文件：`~/.opentrad/opentrad.db`（macOS/Linux）或 `%APPDATA%\OpenTrad\opentrad.db`（Windows）。

用 `better-sqlite3`（同步 API，不用 async/await，在主进程里够用）。

### Schema

```sql
-- 会话元数据（不存完整对话内容，对话内容在 CC 自己的 ~/.claude/projects/ 里）
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,              -- UUID
  title TEXT NOT NULL,
  skill_id TEXT,                    -- 关联的 skill
  cc_session_path TEXT,             -- ~/.claude/projects/<encoded-cwd>/<session>.jsonl
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  last_model TEXT,                  -- 最后使用的模型
  total_cost_usd REAL DEFAULT 0,
  message_count INTEGER DEFAULT 0,
  status TEXT CHECK(status IN ('active', 'completed', 'cancelled', 'error'))
);

CREATE INDEX idx_sessions_updated ON sessions(updated_at DESC);
CREATE INDEX idx_sessions_skill ON sessions(skill_id);

-- 事件缓存（用于离线回放，CC 原始 NDJSON 事件）
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL,
  seq INTEGER NOT NULL,             -- 事件在 session 内的序号
  type TEXT NOT NULL,
  payload TEXT NOT NULL,            -- JSON
  timestamp INTEGER NOT NULL,
  FOREIGN KEY (session_id) REFERENCES sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_events_session_seq ON events(session_id, seq);

-- Risk Gate 规则（用户的"永远允许"选择）
CREATE TABLE risk_rules (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  skill_id TEXT,                    -- NULL 表示适用所有 skill
  tool_name TEXT,                   -- NULL 表示适用所有工具
  business_action TEXT,
  decision TEXT NOT NULL CHECK(decision IN ('allow', 'deny')),
  created_at INTEGER NOT NULL
);

CREATE UNIQUE INDEX idx_risk_rules_key ON risk_rules(
  COALESCE(skill_id, ''),
  COALESCE(tool_name, ''),
  COALESCE(business_action, '')
);

-- 审计日志（per session，查询频率低，不建太多索引）
CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp INTEGER NOT NULL,
  session_id TEXT NOT NULL,
  skill_id TEXT,
  tool_name TEXT NOT NULL,
  business_action TEXT,
  params_json TEXT,                 -- 脱敏后的参数
  decision TEXT NOT NULL,
  automated INTEGER NOT NULL,       -- 0=user, 1=auto
  reason TEXT
);

CREATE INDEX idx_audit_session ON audit_log(session_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);

-- 设置（key-value）
CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,              -- JSON 序列化
  updated_at INTEGER NOT NULL
);

-- 已安装的第三方 skill
CREATE TABLE installed_skills (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL CHECK(source IN ('builtin', 'user_import', 'marketplace')),
  version TEXT NOT NULL,
  install_path TEXT NOT NULL,
  enabled INTEGER NOT NULL DEFAULT 1,
  installed_at INTEGER NOT NULL
);
```

### 文件系统布局

用户数据目录：

```
~/.opentrad/                        # macOS/Linux
├── opentrad.db                     # SQLite 主数据库
├── ipc.sock                        # Unix socket
├── drafts/                         # skill 生成的草稿
│   ├── 2026-04-24-rfq-shenzhen-abc.md
│   └── 2026-04-24-email-quotation.md
├── skills/                         # 用户导入的第三方 skill
│   └── my-custom-skill/
├── logs/                           # 运行日志
│   └── 2026-04-24.log
└── tmp/                            # 临时文件（mcp-config 等）

# Windows 版本
%APPDATA%\OpenTrad\                 
```

**关键原则**：我们不碰 `~/.claude/` 目录，那是 Claude Code 的地盘。

---

## 六、UI 架构

### 路由结构

```
/                       # 主工作区（当前任务对话）
/skills                 # Skill 市场
/skills/:id             # Skill 详情
/history                # 历史任务列表
/history/:sessionId     # 历史任务详情
/settings               # 设置
/settings/cc            # CC 设置子页
/settings/mcp           # MCP 管理子页
/settings/risk          # Risk Gate 设置子页
/onboarding             # 首次引导（独立于主布局）
```

### 组件分层

```
App
├── OnboardingGate        # 检查是否首次启动，是则重定向到 onboarding
├── Routes
│   ├── MainLayout        # 三栏布局
│   │   ├── Sidebar       # 左栏：skill 列表 + 历史
│   │   ├── MainWorkArea  # 中栏：当前任务
│   │   │   └── Chat
│   │   │       ├── MessageList
│   │   │       │   ├── UserMessage
│   │   │       │   ├── AssistantMessage
│   │   │       │   ├── ThinkingBlock
│   │   │       │   ├── ToolCallCard
│   │   │       │   └── ToolResultCard
│   │   │       └── InputBox
│   │   └── RightPanel    # 右栏：辅助信息（浏览器预览、任务进度）
│   └── OnboardingPage    # 首次引导页面（独立）
└── GlobalDialogs
    ├── RiskGateDialog    # Risk Gate 确认弹窗
    ├── ErrorDialog
    └── UpdateAvailableDialog
```

### Zustand stores 设计

```typescript
// stores/session.ts
interface SessionStore {
  currentSessionId: string | null;
  currentMessages: Message[];
  activeSessions: Map<string, SessionMeta>;
  isRunning: boolean;
  
  startTask(skillId: string, inputs: Record<string, any>): Promise<void>;
  cancelCurrent(): Promise<void>;
  resumeSession(sessionId: string): Promise<void>;
  sendMessage(text: string): Promise<void>;
}

// stores/skill.ts
interface SkillStore {
  builtinSkills: SkillManifest[];
  userSkills: SkillManifest[];
  enabledSkillIds: Set<string>;
  
  loadSkills(): Promise<void>;
  enableSkill(id: string): Promise<void>;
  disableSkill(id: string): Promise<void>;
}

// stores/settings.ts
interface SettingsStore {
  language: 'zh-CN' | 'en';
  theme: 'light' | 'dark' | 'system';
  ccStatus: CCStatus;
  riskGateDefaults: RiskGateDefaults;
  
  load(): Promise<void>;
  update(patch: Partial<SettingsState>): Promise<void>;
}

// stores/ui.ts
interface UIStore {
  sidebarCollapsed: boolean;
  rightPanelOpen: boolean;
  activeModal: 'risk-gate' | 'error' | null;
  
  toggleSidebar(): void;
  setRightPanelOpen(open: boolean): void;
  openModal(modal: string): void;
}
```

---

## 七、关键流程的序列图（文字描述）

### 7.1 首次启动流程

```
用户打开 OpenTrad
  ↓
Main: 读取 settings, 检查 "onboarded" 标志
  ↓
未 onboard → Renderer 导航到 /onboarding
  ↓
OnboardingStep1: 检测 CC
  ├─ detectInstallation() → installed=false
  │   ↓
  │   显示"一键安装"按钮 → 在内嵌 terminal 跑 install 脚本
  │   ↓
  │   轮询 detectInstallation() 直到 installed=true
  │
  └─ installed=true → 进入 Step 2
  ↓
OnboardingStep2: 检测登录
  ├─ getAuthStatus() → loggedIn=false
  │   ↓
  │   显示"登录 Claude"按钮 → 在内嵌 terminal 跑 claude auth login
  │   ↓
  │   轮询 getAuthStatus() 直到 loggedIn=true
  │
  └─ loggedIn=true → 进入 Step 3
  ↓
OnboardingStep3: 选择首个 skill
  ↓
用户选择 → 标记 onboarded=true → 导航到主界面
```

### 7.2 启动任务流程

```
用户在 SkillPicker 选了 trade-email-writer → 打开输入表单
  ↓
用户填表 → 点"发送"
  ↓
[Renderer] sessionStore.startTask('trade-email-writer', inputs)
  ↓
[Renderer → Main IPC] cc:start-task { skillId, inputs }
  ↓
[Main] 处理:
  1. SkillLoader.load(skillId) → manifest
  2. PromptComposer.compose(manifest, inputs) → fullPrompt
  3. McpConfigWriter.generateForSession(sessionId, manifest) → configPath
  4. CCManager.startTask({ sessionId, prompt, mcpConfigPath, allowedTools, ... })
      ↓
      spawn claude CLI
      ↓
      返回 CCTaskHandle
  5. 订阅 handle.events，每个 event 通过 IPC 推送给 Renderer
  6. 写 session 元数据到 SQLite
  ↓
[Renderer] 收到 cc:event 事件，更新 UI
  ↓
[循环] 直到 type='result' 事件
  ↓
[Main] 收到 result → 更新 session status='completed' → 清理临时文件
```

### 7.3 Risk Gate 拦截流程

```
CC 决定调用 mcp__opentrad__send_rfq（假设这是高风险工具）
  ↓
[CC → MCP server via stdio] tool call
  ↓
[MCP server] 路由到 tool handler
  ↓
[MCP server] RiskGate middleware: tool.riskLevel='review'
  ↓
[MCP server → Main via IPC bridge] risk-gate.request { tool, params, skill, ... }
  ↓
[Main] RiskGate.check()
  1. 查 risk_rules 表 → 无匹配规则
  2. [Main → Renderer IPC] risk-gate:confirm { req }
  ↓
[Renderer] 展示 RiskGateDialog
  ├─ 展示工具名、参数（脱敏）、skill 名、业务动作
  └─ 按钮: [允许一次] [以后都允许] [拒绝] [编辑后再发]
  ↓
用户点"允许一次"
  ↓
[Renderer → Main IPC] risk-gate:response { decision: 'allow_once' }
  ↓
[Main] 写 audit_log → 返回 decision
  ↓
[Main → MCP server via IPC bridge] response { allow }
  ↓
[MCP server] 执行 tool → 返回结果给 CC
```

---

## 八、架构决策记录（ADR）

### ADR-001：选择 Electron 而非 Tauri

**决策**：v1 使用 Electron 34+。

**理由**：
1. 核心功能依赖 `node-pty`（包装 CC CLI）和 `xterm.js`（terminal 渲染），这两个在 Node 生态里是事实标准；Tauri 要用 Rust sidecar 才能实现，多一层复杂度。
2. Claude Code 施工时 TS/Node 样板代码比 Rust 多几个数量级。
3. 发布链路（签名、自动更新、三平台打包）Electron 更成熟。
4. 反正 CC 自己是 native binary，Electron 带 Chromium + Node 的体积代价（~150MB）不影响核心定位。

**代价**：
- 应用体积比 Tauri 大 100MB+
- 内存占用高（每个渲染进程一份 Chromium）

**回滚条件**：v2 或 v3 如果用户反馈强烈要求"更轻量"，可考虑把内核重写为 Tauri（核心代码大部分是 TS，可复用）。

### ADR-002：monorepo 而非多 repo

**决策**：用 pnpm workspaces 的 monorepo。

**理由**：
1. `desktop` / `mcp-server` / 多个 `packages/*` 需要共享 TS 类型定义和工具函数。
2. 单一 commit 能改跨包的接口变更。
3. CI 只需要一个 workflow。

**代价**：首次 clone 体积大一点。

### ADR-003：Zustand 而非 Redux/Jotai

**决策**：状态管理用 Zustand 5。

**理由**：
1. API 简单，对 TS 原生友好，无模板代码。
2. 适合中等规模应用（OpenTrad v1 规模估算 10-15 个 store）。
3. 不需要 Redux 那套中间件生态（我们的 async 都在主进程做）。

**回滚条件**：store 数量超过 30 个、跨 store 协作复杂时，考虑迁移到 Redux Toolkit。

### ADR-004：better-sqlite3 而非 prisma/drizzle

**决策**：直接用 better-sqlite3 的裸 API，不加 ORM。

**理由**：
1. Schema 稳定简单（7 张表），不需要迁移工具。
2. ORM 在 Electron 主进程里有启动成本（Prisma 的 query engine 要额外 bundle）。
3. 同步 API 省去 async/await 复杂度。

**代价**：写 SQL 字符串，没有类型安全。

**缓解**：用 `zod` 做 row 的运行时校验，手写类型定义。

### ADR-005：shadcn/ui 而非 Ant Design/MUI

**决策**：UI 用 shadcn/ui（复制组件源码）+ Tailwind 4。

**理由**：
1. shadcn 不是 npm 包，是"复制到你项目的可修改源码"，符合开源项目"一切可改"的气质。
2. 依赖轻（只依赖 Radix UI primitives）。
3. Tailwind 风格现代，不像 Ant/MUI 有强烈的"中国政务"或"Material"视觉烙印。
4. Claude Code 对 shadcn 熟悉度很高（训练数据覆盖充分）。

**代价**：组件比成品库少，复杂组件（datatable、chart）要额外找。

### ADR-006：MCP server 独立进程而非 in-process

**决策**：OpenTrad 的 MCP server 作为独立 Node.js 进程运行（CC 通过 stdio 拉起）。

**理由**：
1. CC 的 MCP 协议默认就是"host 拉起 server 进程"模式，不需要改 CC 行为。
2. 隔离性：MCP server 里跑 Playwright（浏览器自动化）崩了不影响 desktop 主进程。
3. 可以独立打包、独立升级。

**代价**：需要 IPC bridge 和 desktop 主进程通信（用 Unix socket/named pipe）。

**可能的替代**：等 Anthropic 推 HTTP transport for MCP 稳定后，可以改成 HTTP 本地 server，调试更方便。

### ADR-007：浏览器自动化用 Playwright 而非 Puppeteer 或裸 CDP

**决策**：用 Playwright 1.5+。

**理由**：
1. API 比 Puppeteer 更现代（自动等待、locator 抽象）。
2. 支持 Chromium / Firefox / WebKit 三内核（未来想做的话）。
3. 官方维护质量高。
4. Accio 调研里看到它内部用 CDP + relay 是因为要对接用户已有 Chrome，我们 v1 不做那个，走 Playwright 自带 Chromium 更简单。

**代价**：Playwright 首次运行要下载 ~150MB Chromium。**缓解**：我们打包时 bundle 进去，用户零等待。

**v2 考虑**：如果用户强烈要求"用我自己的 Chrome"，加一个"接管模式"（CDP 9222）作为可选路径。

### ADR-008：i18n 文案走 JSON 而非 gettext

**决策**：i18next + JSON 文件。

**理由**：代码里写 `t('sidebar.skills.title')`，文案放 JSON。比 gettext/po 简单，GitHub PR 审阅友好。

### ADR-009：日志和错误上报

**决策**：
- 运行日志：写本地 `~/.opentrad/logs/YYYY-MM-DD.log`，按日切割，保留 7 天。
- 错误上报：**不自动上报**。用户主动点"报 bug"才收集并打开 GitHub Issue 预填页面。

**理由**：隐私第一，外贸商家对"数据上报"极度敏感。

### ADR-010：版本号与发布策略

**决策**：
- 遵循 SemVer 2.0.0
- v0.1.x / v0.5.x 为 beta 阶段，允许 breaking change
- v1.0.0 之后严格 SemVer
- 发布渠道：`stable` / `beta`，用户可选
- 自动更新：electron-updater，只在用户同意下更新

---

## 九、安全和隐私

### 数据流审计

**离开用户本机的数据**：
1. 给 Claude Code 的请求（通过 CC → Anthropic 服务器）——这是用户自己的订阅，OpenTrad 不增加额外流量
2. 给第三方 MCP server 的请求（如果用户自己配了）——用户知情
3. `web_fetch` / `browser_open` 到目标网站——任务必需

**绝对不离开本机的数据**：
1. 用户的 Claude 凭证（我们不读）
2. 任务内容（CC 有副本在 `~/.claude/`，但不走 OpenTrad 的服务器）
3. 崩溃日志（除非用户主动"报 bug"）
4. 使用统计（v1 完全不做 telemetry）

### 凭证管理

- **OpenTrad 自己的**：无（v1 没有账号系统）
- **Claude 的**：CC 管，存 Keychain，我们不读
- **用户第三方 API key**（v1.5+）：加密存储在 SQLite，用 `keytar` 保护主密钥

### 沙箱和权限

- Electron 开启 contextIsolation 和 sandbox
- MCP server 进程用有限权限启动（只能访问 `~/.opentrad/` 和用户指定的项目目录）
- Risk Gate 作为业务级兜底，在 CC 的 permissionMode 之上

---

## 十、开发工作流

### 目录惯例

- 所有 TS 文件默认 `.ts` 或 `.tsx`
- React 组件用 PascalCase：`MessageBubble.tsx`
- 非组件文件用 kebab-case：`cc-manager.ts`
- 类型文件 `types.ts` 或 `*.types.ts`
- 测试文件 `*.test.ts`（vitest）或 `*.e2e.ts`（playwright）

### 提交规范

Conventional Commits：
- `feat: add 1688 search tool`
- `fix: stream parser crashes on empty lines`
- `docs: update install guide`
- `chore: bump electron to 34.1`

### PR 流程

v1 单人开发阶段：直接 push main（但带 pre-commit hook 跑 Biome check）。

v2 开放贡献后：feature branch → PR → CI 通过 → BDFL 合并。

### CI/CD

GitHub Actions workflow：

```yaml
# .github/workflows/ci.yml
- lint（Biome）
- typecheck（tsc --noEmit）
- test（vitest）
- build desktop（electron-builder，三平台 matrix）
- artifact upload（所有 tag 自动发 GitHub Release）
```

---

## 十一、性能目标

| 指标 | 目标 |
|------|------|
| 冷启动时间 | ≤ 3 秒（从点击图标到主界面） |
| CC spawn 到首个事件 | ≤ 2 秒 |
| UI 响应（点击 → 反馈） | ≤ 100ms |
| 内存占用（idle） | ≤ 300MB（不算 CC） |
| 内存占用（单任务跑） | ≤ 500MB（不算 CC） |
| 应用体积（macOS .dmg） | ≤ 300MB（含 Chromium） |

---

## 十二、未来扩展预留

虽然 v1 不做，但架构要留口子：

1. **v1.5 Codex CLI 宿主**：`cc-adapter` 抽象成 `agent-adapter` 接口，CC 是其中一个实现，Codex 是另一个。
2. **v1.5 国产模型 BYOK**：`cc-adapter` 接口加一个 `direct-api` 实现（不走 CC，直接走 API），走 OpenAI-compatible schema。
3. **v2 企业版**：主进程加一层"远程审批"handler，当前的 RiskGate 走 Renderer IPC，将来可以走 WebSocket 到手机 App。
4. **v2 云同步**：SQLite 加一个 sync 层，可选挂载到用户的 GitHub 或自建服务器。
5. **v2 skill marketplace**：Skill Loader 加一个 `fetchFromMarketplace()` 方法，初始版本连到 `github.com/opentrad/skills` 的 GitHub Releases。

