# OpenTrad v1 MVP 功能切片

**文档版本**：1.0  
**撰写日期**：2026-04-24  
**状态**：已审阅
**作者**：总工（Claude Opus 4.7）  
**依赖**：`01-positioning.md` v1.1

---

## 本文档的用途

这份文档是给施工单位（Claude Code）看的**施工图**。定位书讲"为什么 / 做什么"，这份讲"具体做哪些 feature、什么顺序、做成什么样算完成"。

每个功能都会标注：
- **模块归属**：5 大架构模块中的哪一个
- **优先级**：P0（v0.1 必须有）/ P1（v0.3 前有）/ P2（v1.0 前有）
- **验收标准**：怎么才算"做完了"
- **依赖**：需要哪些模块/功能先就位

---

## 一、里程碑规划

v1 拆成 4 个里程碑，每个里程碑都是**可以端到端演示**的版本（不是内部半成品）。

### 里程碑 M0：骨架可跑（内部版）

**目标**：桌面应用能打开，能启动 Claude Code 子进程，能渲染出 hello world 级别的对话。

**验收标准**：
- `pnpm dev` 能启动 Electron 应用
- 应用检测到本机 Claude Code，能 spawn `claude -p --output-format stream-json "Say hi"`
- 流式 NDJSON 事件被正确 parse，assistant 消息渲染到 UI
- 关闭应用时子进程正确清理

**时间预算**：7-10 天

**本里程碑的风险**：`stream-json` 在不同 CC 版本上的事件 schema 可能有差异，要早暴露。

### 里程碑 M1：单 skill 能跑通（v0.1 公开 preview）

**目标**：用户能装上应用、登录 Claude、选一个 skill（比如 `trade-email-writer`）、从头到尾完成一个任务。

**验收标准**：
- 有安装向导，能检测/引导安装 Claude Code
- 有登录引导（内嵌 terminal 让用户跑 `claude auth login`）
- 至少 1 个 skill（`trade-email-writer`）完整可用
- 对话能多轮、能续聊、能导出
- 能打包发 `.dmg` / `.exe` / `.AppImage`

**时间预算**：M0 之后 15-20 天

**这是第一个能发布的版本**，标记为 **v0.1**，在 GitHub Releases 发出来，允许社区试用。

### 里程碑 M2：5 个 skill 齐 + MCP server + Risk Gate（v0.5 Beta）

**目标**：定位书里承诺的 5 个 P0 skill 全部可用，OpenTrad 自己的 MCP server 完整（浏览器工具、1688 工具等），Risk Gate 在关键动作上生效。

**验收标准**：
- 5 个 P0 skill 全部通过端到端测试
- OpenTrad MCP server 暴露的工具集完整
- Risk Gate 在 `send_rfq` / `submit_form` / `oauth_allow` 等动作上触发确认弹窗
- 任务历史 / session 列表可以查看、继续、删除

**时间预算**：M1 之后 25-35 天

**这是 v1 的功能齐版本**，标记为 **v0.5 Beta**，外贸商家能真实使用。

### 里程碑 M3：打磨 + 文档 + 首发（v1.0 GA）

**目标**：性能优化、bug 清理、完整文档、内置教程、中英双语 UI 就绪。

**验收标准**：
- 所有 P2 功能完成（见下文清单）
- 文档站上线（opentrad.dev 或 docs.opentrad.com）
- 内置新手教程（在应用内，不是外部文档）
- 中英文 UI 完整
- CI/CD 自动打包发布（3 平台：macOS / Windows / Linux）
- GitHub Release 页有完整 changelog、安装教程、FAQ

**时间预算**：M2 之后 15-20 天

**v1.0 GA**，正式推广。

### 总预算

**保守估算**：M0 + M1 + M2 + M3 = 62-85 天 ≈ 2-3 个月全职工作量。

考虑到你是兼顾学业和工作的单人团队、借助 Claude Code 施工、前期设计已到位，**预期 v1.0 从本文档完稿起 4 个月内上线**是可以的。激进一点 3 个月也不是不可能。

---

## 二、五大架构模块的功能切片

这里的"模块"对应 codex 调研给出的五模块拆分（Process Manager / Stream Parser / Skill Manifest / MCP Config Writer / Risk Gate）。每个模块下的功能按优先级排列。

### 模块 1：Process Manager（进程管理器）

**模块职责**：管理 Claude Code CLI 子进程的整个生命周期——启动、通信、监控、清理。

#### F1.1 [P0] CC 存在性检测

- 启动时执行 `claude --version`，记录版本号
- 记录版本到 schema 兼容性矩阵（本文档末尾维护）
- 未检测到时触发安装向导（见 F6.1）

**验收**：全新机器上第一次启动，能识别"无 CC"并进入安装流程；已装机器上能识别版本号并展示在设置面板。

#### F1.2 [P0] CC 子进程 spawn 与管理

- 使用 `child_process.spawn` 启动 `claude -p --output-format stream-json --verbose`
- 为每个任务分配唯一 UUID 作为 `--session-id`
- 支持 kill（用户取消任务）、正常结束、异常退出三种终结
- 应用关闭时确保所有子进程被清理

**验收**：能同时启动、识别、清理 3 个以上并发子进程，应用退出后 `ps` 看不到残留 `claude` 进程。

#### F1.3 [P0] 登录状态检测

- 调用 `claude auth status --text`
- parse 输出判断是否已登录、登录方式（Pro / Max / API key）
- 未登录时引导用户进入登录流程（见 F6.2）
- **不读取凭证文件，不搬运 token**

**验收**：能正确识别三种状态（未登录 / Pro 订阅已登录 / API key 已登录），展示账号邮箱（脱敏为 `u***@example.com` 形式）。

#### F1.4 [P1] PTY fallback

- 当非交互模式遇到权限 prompt 或登录 flow 时，切到 PTY 模式
- 使用 `node-pty` 包装，渲染一个内嵌 `xterm.js` 面板
- PTY 面板默认折叠在侧边栏，用户按"打开 terminal"才展开

**验收**：在登录、首次 workspace trust、CC 权限询问这三个场景下，能看到真实的 CC terminal 交互界面。

#### F1.5 [P1] 任务取消与恢复

- UI 上有"停止"按钮，点了 kill 子进程
- 用户可以"继续上次对话"——调用 `claude --resume <session-id>`
- 历史对话列表可以点击恢复

**验收**：从历史对话列表点击"继续"后，新的子进程能 resume 到同一 session，CC 看到完整上下文。

#### F1.6 [P2] 多 session 并行

- 允许同时开多个任务（比如同时跑 3 个选品查询）
- 每个任务有独立的 session、独立的 UI tab
- 资源上限（建议 v1 最多 3 个并发，防止 CC rate limit）

**验收**：3 个并发任务独立运行、独立进度、独立历史。

#### F1.7 [P2] 进程资源监控

- 每个子进程的 CPU / 内存占用展示在状态栏
- 异常占用时告警
- 支持用户手动 kill 异常进程

**验收**：状态栏能看到每个 session 的资源占用，点击异常任务能 kill。

---

### 模块 2：Stream Parser（流解析器）

**模块职责**：解析 Claude Code 的 `stream-json` NDJSON 输出，转换成 UI 可渲染的事件。

#### F2.1 [P0] NDJSON 行解析

- 逐行读取子进程 stdout
- 每行 parse 成 JSON 对象
- 按 `type` 字段分发到不同 handler

**验收**：收到的每个 JSON 事件都被识别，未知 type 被记录到日志但不崩溃。

#### F2.2 [P0] 事件到 UI 消息映射

| CC 事件类型 | UI 渲染 |
|-----------|--------|
| `system/init` | 不显示，用于配置 UI（模型、tools list） |
| `assistant`（text） | 对话气泡，markdown 渲染 |
| `assistant`（thinking） | 可折叠的"思考过程"块 |
| `tool_use` | 工具调用卡片（工具名 + 参数） |
| `tool_result` | 工具结果卡片（可折叠） |
| `result` | 任务结束标记 + cost + usage |
| `rate_limit_event` | 顶部提示条 |

**验收**：对一个典型任务（"帮我写一封 RFQ 邮件"）的完整流，UI 能正确渲染出每一类事件。

#### F2.3 [P0] Markdown 渲染器

- 对 assistant 消息做 markdown 渲染
- 支持：标题、列表、表格、代码块、链接
- 代码块带复制按钮
- 表格可以"导出为 CSV"（v1 用 markdown 表格识别）

**验收**：CC 输出的中文/英文/代码混排内容能正确显示，表格能复制到 Excel。

#### F2.4 [P0] Schema version guard

- 启动时读取 `system/init` 事件的 `claude_code_version`
- 检查是否在已验证兼容列表内（初始列表：`2.1.x`）
- 不兼容版本弹窗提示"未验证版本，功能可能异常，请先反馈"

**验收**：切换到未支持的 CC 版本，启动时能看到提示弹窗，但不阻止使用。

#### F2.5 [P1] 事件回放

- 历史任务的 NDJSON 事件流持久化到本地（比如 SQLite 或 JSONL）
- 用户打开历史任务时，重新渲染所有事件（不是只展示最后结果）

**验收**：关闭应用后重新打开，历史任务的每个工具调用卡片、思考过程都能重现。

#### F2.6 [P2] 事件搜索

- 在历史任务里搜索关键字（"昨天我问的那个关于蓝牙音箱的任务"）
- v1 用简单全文搜索，v2 再考虑向量检索

**验收**：关键词搜索能找到包含该词的任务，跳转到对应 session。

---

### 模块 3：Skill Manifest（技能系统）

**模块职责**：管理 skill 的定义、安装、启用、生成 prompt。

#### F3.1 [P0] Skill Manifest 格式

统一的 YAML + Markdown 格式：

```yaml
# skill.yml
id: trade-email-writer
title: 外贸邮件写作
version: 1.0.0
risk_level: draft_only
description: 生成 RFQ / 报价 / 议价 / 催付等外贸邮件草稿
category: communication
allowed_tools:
  - Read
  - Write
  - WebSearch
disallowed_tools:
  - Bash(*)
  - mcp__*__send*
stop_before:
  - send_email
  - submit_form
inputs:
  - name: context
    type: text
    label: 邮件场景描述
    required: true
  - name: tone
    type: select
    label: 邮件语气
    options: [formal, friendly, firm]
outputs:
  - draft_email
  - subject_line
  - risk_notes
prompt_template: prompt.md
```

配套 `prompt.md`（系统提示模板）和 `examples/`（示例输入输出）。

**验收**：能加载一个符合格式的 skill 包，跑一次 CC 任务，prompt 被正确注入。

#### F3.2 [P0] 内置 skill 加载器

- 应用启动时扫描 `resources/skills/` 目录（打包时内置的 5 个 P0 skill）
- skill 列表展示在侧边栏，用户点击选择
- 选择后打开 skill 的输入表单（根据 manifest 的 `inputs` 生成）

**验收**：5 个 P0 skill 都在侧边栏可见，点击能打开对应表单。

#### F3.3 [P0] Prompt 组合器

- 用户填完表单 + 选中 skill 后，生成完整 prompt：
  - skill manifest 里的 `prompt_template`
  - 用户输入的 `inputs`
  - 工具白名单（通过 `--allowedTools mcp__opentrad__* Read Write ...`）
- 组合结果传给 Process Manager 启动 CC

**验收**：手动读一份 skill，能看到生成的完整 prompt 和 allowedTools，跑出来的 CC 任务只调用白名单内的工具。

#### F3.4 [P1] Skill 市场 UI

- 内置 skill 市场页面（不是外部链接）
- 展示已安装 / 未安装 skill
- "一键启用 / 禁用"功能
- v1 不做第三方 skill 分发，只展示官方 skill

**验收**：市场页能看到所有 5 个 P0 skill + 5 个 P1 skill（P1 可能未打包，显示"即将推出"）。

#### F3.5 [P2] 第三方 skill 导入

- 支持用户从本地文件夹导入 skill 包（zip 或目录）
- 导入时校验 manifest 格式
- 导入的 skill 标记为"第三方"，有红色警告标识

**验收**：能从本地拖入一个 skill zip，解压后加载，在列表里标红显示。

#### F3.6 [P2] Skill 版本管理

- skill 包带版本号
- 应用可以检测 skill 更新（从 opentrad/skills repo）
- 一键升级

**验收**：把本地 skill 改低版本号，能检测到"有更新"，点更新后 skill 被替换。

---

### 模块 4：MCP Config Writer（MCP 配置生成器）

**模块职责**：生成 OpenTrad 自己的 MCP server 配置，并在每次启动 CC 时传入。

#### F4.1 [P0] OpenTrad MCP server（内置）

OpenTrad 本体内嵌一个 MCP server（通过 stdio transport），提供这些工具：

| 工具名 | 功能 |
|-------|------|
| `browser_open` | 用内置 Chromium 或接管用户 Chrome 打开 URL |
| `browser_read` | 读取当前 tab 内容（DOM snapshot） |
| `browser_screenshot` | 截图当前 tab |
| `1688_search` | 1688 选品搜索（浏览器自动化封装） |
| `alibaba_supplier_view` | 读取 Alibaba 供应商页面信息 |
| `listing_template` | 生成平台 listing 模板 |
| `hs_code_lookup` | HS code 查询（调用中国海关公开接口） |
| `draft_save` | 保存草稿到本地 `drafts/` 目录 |

每个工具在调用前经过 Risk Gate（见模块 5）。

**验收**：CC 通过 `mcp__opentrad__1688_search` 能调用到 1688 搜索工具，返回结构化结果。

#### F4.2 [P0] 运行时 mcp-config 生成

- 每次启动 CC 任务时生成 `opentrad-<session>.mcp.json`
- 传 `--mcp-config <path>` 和 `--strict-mcp-config`
- 任务结束后清理临时文件

**验收**：查看临时目录能看到生成的 mcp config，CC 启动后通过 `/mcp` 命令能看到 opentrad 的工具。

#### F4.3 [P1] 第三方 MCP server 管理

- 应用内有"MCP 管理"页面
- 用户可以添加第三方 MCP server（比如用户自己的 Gmail MCP、Notion MCP）
- 配置写到用户的 `.claude.json` 的 user 级（不污染项目级）

**验收**：用户加一个 echo MCP server，启动 CC 后能在 `/mcp` 里看到并调用。

#### F4.4 [P2] MCP 工具使用统计

- 记录每个 MCP 工具的调用次数、成功率、平均耗时
- 展示在"工具使用报告"页面
- 帮助用户发现哪些工具最有用、哪些总是失败

**验收**：跑一周后，统计页面能看到准确的工具调用数据。

---

### 模块 5：Risk Gate（风险闸）

**模块职责**：在 Claude Code 调用高风险工具前拦截并要求用户确认，这是 OpenTrad 相对 vanilla CC 的核心差异化。

#### F5.1 [P0] 工具级拦截

- OpenTrad MCP server 的每个工具上都有 `risk_level` 标签
- 调用前根据标签决定：
  - `safe` → 直接放行
  - `review` → 弹窗展示参数，用户确认
  - `blocked` → 永远不允许（比如"真实发送邮件"在 v1 blocked）

**验收**：调用 `browser_open` 直接跑通；调用 `draft_save_to_inbox` 弹窗要求确认；调用（虚构的）`send_real_email` 被拒绝。

#### F5.2 [P0] Skill 级 `stop_before` 支持

- skill manifest 里的 `stop_before` 列表里的动作
- 到达这些动作时，CC 会被"暂停"（通过 MCP 工具返回"需要用户确认"的特殊响应）
- UI 弹一个**业务级确认卡片**（不是工具级）：
  > "即将发送 RFQ 给供应商 ShenZhen ABC。收件人：abc@example.com。内容预览：..."
  > [确认发送] [取消] [编辑后再发送]

**验收**：跑 `supplier-rfq-draft` skill，在"发送"这一步 UI 弹出业务级卡片，不是工具级卡片。

#### F5.3 [P1] 审计日志

- 所有被拦截的动作、用户的确认/拒绝、拒绝原因记录到 `audit.jsonl`
- 用户可以在"审计日志"页面查看历史
- 日志字段：timestamp、skill、tool、action、decision、params（脱敏）

**验收**：做 10 个操作后，审计日志有对应 10 条记录，字段齐全。

#### F5.4 [P2] 自定义风险规则

- 用户可以在设置里调整 Risk Gate 策略
- 比如：把某个 skill 的 `browser_open` 从 `review` 降到 `safe`（信任它了）
- 规则按"skill + tool + 动作"三元组

**验收**：用户能创建规则"对 `product-sourcing-1688` skill，`browser_open` 直接放行"，后续调用不再弹窗。

---

## 三、通用功能（不属于 5 模块）

### F6.1 [P0] 安装向导

第一次打开应用的引导流程：

```
欢迎来到 OpenTrad

第 1 步：检查 Claude Code
  ○ 未检测到 Claude Code
  [一键安装]  [我自己装]

第 2 步：登录 Claude
  ○ 未登录
  [打开登录窗口]

第 3 步：选择你的第一个 skill
  ● 1688 选品    ○ 外贸邮件    ○ 商品上架    ○ 供应商询盘    ○ HS code 查询

  [跳过引导]  [开始使用]
```

安装 CC 走官方 install script（调 `curl -fsSL https://claude.ai/install.sh | bash`，在内嵌 terminal 里执行，用户能看到过程）。

**验收**：全新 macOS 机器上从下载 OpenTrad 到完成第一个任务，总共点击次数 ≤ 10 次。

### F6.2 [P0] 登录引导

- UI 上有"登录 Claude"按钮
- 点击后弹出内嵌 terminal，自动执行 `claude auth login --claudeai`
- 用户在 CC 提示下复制 URL 去浏览器登录
- 登录完成后自动检测状态、关闭 terminal、回到主界面

**验收**：按向导走完，主界面能看到"已登录 u***@example.com（Claude Pro）"。

### F6.3 [P0] 主界面布局

```
┌─────────────┬────────────────────────┬──────────────┐
│ 左侧栏      │ 主工作区                │ 右侧辅助栏   │
│ (220px)     │                        │ (280px)      │
│             │                        │              │
│ Skill 列表  │ 对话流                  │ 任务进度     │
│ 历史任务    │                        │ 工具调用详情 │
│ 设置        │ [输入框]               │ 浏览器预览   │
│             │                        │              │
└─────────────┴────────────────────────┴──────────────┘
```

右侧辅助栏在需要时才展开（比如有 browser tool 调用时自动展开浏览器预览）。

**验收**：三栏布局可拖动调整宽度，各自滚动独立。

### F6.4 [P0] 输入框与发送

- 主工作区底部有输入框
- 支持多行、粘贴图片、拖文件
- 发送按钮 + Enter（可配置 Enter/Cmd-Enter）
- 发送后立即清空、显示"思考中..."

**验收**：贴一张 1688 商品截图 + 一段文字描述，能正确发给 CC，CC 能 see_image 看到图。

### F6.5 [P0] 对话历史

- 主界面显示当前 session 的完整对话
- 历史任务侧边栏列出所有 session（标题 / skill / 时间）
- 点击历史任务加载到主界面

**验收**：历史任务按时间倒序排列，点击后对话正确加载。

### F6.6 [P0] 设置面板

最低限度的设置项：

- **一般**：语言（中/英）、主题（浅/深/跟随系统）
- **Claude Code**：版本信息、登录状态、安装路径
- **MCP**：第三方 MCP server 管理（P1）
- **Risk Gate**：默认风险策略
- **关于**：版本、GitHub 链接、许可证

**验收**：设置项全部可打开、可保存、重启后持久化。

### F6.7 [P0] 导出

- 对话可以"导出为 markdown"
- 表格数据可以"导出为 CSV"
- 草稿文件（在 `~/OpenTrad/drafts/`）可以用 Finder/Explorer 打开

**验收**：导出的 markdown 和 CSV 在外部工具里能正常查看。

### F6.8 [P1] 中英文 UI 切换

- 所有 UI 文案走 i18n
- 默认跟随系统语言
- 可在设置手动切换

**验收**：设置里切换语言后，所有静态文案立即更新。

### F6.9 [P1] 内置新手教程

- 第一次打开或在设置里可以触发
- 逐步引导用户走完：选 skill → 填表单 → 发送 → 看结果 → 导出
- 像产品 tour 那种高亮 + 箭头

**验收**：新用户跟着教程走完能完成一次完整任务。

### F6.10 [P2] 错误上报

- 用户可以一键"报告 bug"
- 自动收集：CC 版本、OpenTrad 版本、OS、最近的错误日志
- **脱敏**：邮箱 / token / 路径中的用户名
- 生成 GitHub Issue 预填链接（不上传任何数据到 OpenTrad 服务器）

**验收**：点"报告 bug"打开 GitHub Issue 创建页，预填内容完整，敏感信息已脱敏。

---

## 四、v1 五个 P0 Skill 的详细验收标准

每个 skill 都要有明确的"做到什么程度算完成"，否则工程时会越做越大。

### Skill 1：`product-sourcing-1688`

**输入**：
- 需求描述（中文自然语言，如"找 10 款适合东南亚夏季的蓝牙音箱，预算 30 元内"）
- 最多候选商品数（默认 10）
- 优先指标（默认"价格 + MOQ"，可选"销量"、"店铺等级"）

**执行流程**：
1. 从需求描述提取搜索词和筛选条件
2. 通过 `1688_search` MCP 工具搜索
3. 对每个候选调用 `browser_read` 读取详情页
4. 提取：标题 / 价格 / MOQ / 销量 / 店铺信用 / 关键参数
5. 生成对比表格

**输出**：
- Markdown 表格（商品名 / 价格 / MOQ / 销量 / 供应商 / 链接）
- 每款商品的风险提示（"MOQ 过高"、"店铺新"、"描述与图片不符"等）
- 推荐 top 3 的理由

**验收**：
- 给定测试需求，能返回至少 5 个真实存在的 1688 商品
- 表格字段齐全，链接可点击
- 至少 2 条风险提示有价值（不是套话）
- 端到端时长 ≤ 3 分钟

**禁止**：自动加购物车、自动联系供应商。

### Skill 2：`supplier-rfq-draft`

**输入**：
- 供应商 Alibaba 页面 URL
- 询盘意图（示例："我想采购 500 件、需要 OEM logo、12 月出货"）

**执行流程**：
1. 访问供应商页面，提取：公司名 / 主营 / 认证 / MOQ / 交付时间 / 联系方式
2. 分析买家意图，生成 RFQ 草稿（英文）
3. 列出"需要供应商澄清的问题清单"
4. 列出"买家自己需要补充的信息"（比如"你有目标 FOB 价格吗"）

**输出**：
- RFQ 邮件草稿（subject + body）
- 需澄清问题清单（5-10 条）
- 买家信息补充清单
- 供应商风险评估（认证 / 年份 / 差评）

**验收**：
- 草稿英文地道（不是翻译腔），包含专业术语（FOB / CIF / MOQ / lead time）
- 问题清单切实有用（不是"请告诉我价格"这种废话）
- 供应商风险评估至少 3 条有依据的观察
- **绝对不发送**（没有"发送"按钮，只有"复制"/"导出"）

### Skill 3：`trade-email-writer`

**输入**：
- 场景类型（报价 / 议价 / 催付 / 催货 / 售后 / 跟进）
- 上下文（与谁通信、之前发生了什么、目的）
- 语气（formal / friendly / firm）
- 可选：上一封邮件原文（用于回复场景）

**执行流程**：
1. 根据场景类型调用对应 prompt 模板
2. 结合上下文生成邮件
3. 提供 2-3 个不同角度的版本供选择

**输出**：
- 2-3 封邮件草稿（subject + body），每封注明"策略角度"
- 关键短语解释（某些行业术语为什么这么写）
- 风险提示（比如"这封信的语气已经接近最后通牒，再激烈可能导致对方关系破裂"）

**验收**：
- 生成的邮件符合外贸行业惯例（问候 / 正文 / 行动项 / 结尾得体）
- 不同语气选项真的有区别（formal vs friendly 读起来明显不同）
- 不出现"作为 AI 助手"类开场白
- **不发送**，只导出 / 复制

### Skill 4：`listing-localizer-seo`

**输入**：
- 商品中文资料（标题 / 描述 / 参数 / 卖点）
- 目标平台（Amazon / Shopify / TikTok / Alibaba）
- 目标市场（US / UK / EU / SEA）

**执行流程**：
1. 根据目标平台规范（字数 / 格式要求）分块处理
2. 生成英文标题（含 SEO 关键词）
3. 生成五点描述（Amazon 风格）或商品详情（其他平台）
4. 提取关键词（search terms / tags）
5. 给出合规性提示（不合规表述、禁用词）

**输出**：
- 英文标题（多个候选）
- 主描述（按平台格式）
- 关键词清单（含中英文对照）
- 合规性提示（如"US 市场禁用 'cure'、'treat' 等医疗暗示词"）

**验收**：
- 标题不超过平台字数限制（Amazon 200 字符 / TikTok 100 字符等）
- 关键词有流量判断依据（简单调用 web_search 看竞品）
- 合规性提示至少 1 条针对目标市场
- 导出格式可以直接贴到平台后台

### Skill 5：`hs-code-compliance-check`

**输入**：
- 商品描述（中文）
- 目标市场（中国出口目的地）

**执行流程**：
1. 根据商品描述推断 HS code（通过 `hs_code_lookup` 工具）
2. 给出最可能的 1-3 个 HS code 候选
3. 查询出口禁限售目录
4. 查询目标国进口合规要求（认证 / 标签 / 关税）
5. 生成报关英文描述

**输出**：
- HS code 候选（附置信度 + 理由）
- 中国出口合规状态（正常 / 禁限售 / 许可证）
- 目标国合规要求清单
- 报关英文描述（标准格式）
- **明确声明**："本结果仅供参考，以海关认定为准，复杂品类请咨询报关行"

**验收**：
- HS code 推断对常见品类（蓝牙音箱 / 服装 / 化妆品）准确
- 合规提示包含认证要求（CE / FCC / 目标国特定）
- 免责声明显著（不是小字隐藏）
- 不提供"如何规避"类建议（防止法律风险）

---

## 五、不做 / 不支持 列表

明确写下 v1 不做的东西，避免施工过程功能蔓延。

**不做的功能**：

- ❌ 账号系统（登录 OpenTrad 本身）
- ❌ 云端同步、多设备
- ❌ 团队协作、评论、@
- ❌ 真实发送邮件 / IM / 任何对外通信
- ❌ 自动下单、自动付款
- ❌ 自动发布到任何平台
- ❌ 自研 agent runtime
- ❌ 接 Codex / 国产模型 / API key 直连（v1 只做 CC 宿主）
- ❌ 向量记忆 / 长期记忆（v1 只用 CC 自己的 session 持久化）
- ❌ 手机端 / Web 端

**不支持的环境**：

- ❌ 32-bit 系统
- ❌ ARM 之外的老 Mac（v1 Apple Silicon + Intel 64-bit）
- ❌ Windows 10 以下（v1 仅 Win 10 21H2 及以上）
- ❌ Linux 无 GUI 环境（v1 需要 X11 或 Wayland）

---

## 六、CC 版本兼容性矩阵

Stream Parser schema 会随 CC 升级变化。维护一张矩阵：

| CC 版本 | stream-json 状态 | 备注 |
|--------|-----------------|------|
| 2.1.0 - 2.1.x | ✅ 已验证 | 初始支持版本 |
| 2.2.x | ⏳ 发布后测试 | |
| 2.0.x 以下 | ❌ 不支持 | 缺少 `--mcp-config` 等关键参数 |

每次 CC 升级 minor 版本时，跑一遍回归测试；每次 major 版本升级时，评估是否需要适配层。

---

## 七、功能清单汇总表

按里程碑汇总的功能数量：

| 里程碑 | P0 | P1 | P2 | 本阶段总计 | 累计 |
|-------|-----|-----|-----|-----------|-----|
| M0 骨架可跑 | 6 | 0 | 0 | 6 | 6 |
| M1 v0.1 preview | 12 | 0 | 0 | 12 | 18 |
| M2 v0.5 Beta | 5 | 7 | 0 | 12 | 30 |
| M3 v1.0 GA | 0 | 3 | 10 | 13 | 43 |

全部 P0 功能共 23 个，P1 共 10 个，P2 共 10 个。总计 43 个 issue 级功能。

---

## 八、依赖关系图

关键依赖（谁依赖谁）：

```
F1.1 CC检测 ─┐
F1.2 spawn ──┴─► F2.1 NDJSON 解析 ─► F2.2 事件渲染
F1.3 登录 ───────► F6.2 登录引导

F3.1 Manifest ─► F3.3 prompt 组合器 ─┐
F4.1 MCP server ─────────────────────┴─► 完整任务流

F5.1 工具拦截 ─┐
F5.2 skill 级 ─┴─► Risk Gate 完整

F6.1 安装向导 ─► 所有上面的 (没装上啥也没用)
```

M0 阶段必须完成的最小集：F1.1 / F1.2 / F2.1 / F2.2 / F2.3 / F6.3 主界面外壳。