# OpenTrad M0 回顾

**文档版本**：1.0  
**撰写日期**：2026-04-24  
**作者**：总工（Claude Opus 4.7）  
**M0 起止**：2026-04-24 开工 → 2026-04-24 收官  
**Tag**：[`m0-complete`](https://github.com/open-trad/opentrad/releases/tag/m0-complete) (`8748590112...`)

---

## 一、TL;DR

M0 在**预算 12-18 天**的情况下**实际 3-4 天完成**。8 个 issue 全部 close、8 个 PR 全部 merge、CI 三平台自动守门、真实 Claude Code 2.1.119 端到端验证通过。Hello World demo 跑出 4.6 秒延迟、$0.009503 cost。

执行速度远超预期的根本原因有三个：

1. **设计三件套（01/02/03）的抽象层次准确**——施工过程中所有澄清都聚焦在"文档没覆盖的真实细节"，没有"文档我看不懂"
2. **Claude Code 作为唯一施工单位的协作模式有效**——总工设计、发起人决策、Claude Code 施工的三角分工降低了沟通成本
3. **决策颗粒度的精细化**——D1/D6/D9 三次架构级拍板都基于真实数据/实测样本/具体代价，不是空中设计

但执行速度也暴露了三件套的**真实质量问题**：架构文档 §4.2 共出现两次重大勘误（字段命名、扁平化决策），这两次都是 Issue #4 抓真实 CC 样本时被动发现的。如果文档撰写时对着真实 CC 输出校对一次，能省 1-2 天。**M1 撰写架构文档时必须先抓真实数据**。

---

## 二、时间预算 vs 实际

### 原预算（来自 04-kickoff-checklist.md）

| Issue | 预估 |
|-------|------|
| #1 monorepo 骨架 | 1 天 |
| #2 shared 类型 | 1-2 天 |
| #3 stream-parser | 2-3 天 |
| #4 cc-adapter | 3-4 天 |
| #5 Electron 骨架 | 2 天 |
| #6 IPC | 2 天 |
| #7 Hello World | 2-3 天 |
| #8 CI/CD | 1 天 |
| **合计** | **14-18 天** |

### 实际工作量

| GitHub Issue | PR | 实际跨度 |
|--------------|-----|----------|
| #2 monorepo 骨架 | #10 | Day 1 |
| #3 shared 类型 | #11 | Day 1 |
| #4 stream-parser | #12 | Day 1-2 |
| #5 cc-adapter | #13 | Day 2 |
| #9 CI/CD | #14 | Day 2-3 |
| #6 Electron 骨架 | #15 | Day 3 |
| #7 IPC | #16 | Day 3 |
| #8 Hello World | #17 | Day 3-4 |
| **合计** | | **3-4 天** |

注：GitHub Issue 编号比文档 Issue 编号偏 1（GitHub #2 = 文档 Issue #1，因为 Issue #1 被门面文件 PR 占用）。

### 速度差异原因分析

**原预算偏保守的合理原因**：
- 包含了"工程师不熟悉技术栈"的学习时间——Claude Code 不需要
- 包含了"问题需要异步沟通"的等待时间——Claude Code 同步沟通
- 包含了"上下文切换"的损耗——Claude Code 持续 focus

**实际执行优化的关键点**：
- D1/D6/D9 三次架构决策都是**实战中即时拍板**，不是先设计后实现
- Issue #2/#3/#4/#5 早期 4 个 issue 在 Day 1 一天内连续推进，因为它们都是"按文档实现"的纯施工任务
- 后期 #6/#7/#8 因为有 UI 工作和真实 CC 端到端联调略慢，但 IPC 协议提前定义清晰避免了返工

**M1 时间预算建议**：
- 不要按 M0 的 3-4 天速度乐观推算 M1。M1 涉及 skill runtime / MCP server / browser tools / Risk Gate 四个新模块，复杂度比 M0 高 2-3 倍
- M1 issue 数量预计 12 个，但每个 issue 平均工作量比 M0 大
- 合理预算：**M1 7-12 天**（02-mvp-slicing.md 原预算 25-35 天偏保守，按 M0 验证过的速度可压缩）

---

## 三、三次架构级决策的追溯

M0 期间 Claude Code 抛出了三次"决策点"，由发起人和总工拍板。这三次都是**架构级决策**——不是实现细节，而是会影响整个 codebase 风格和后续可维护性的选择。

### D1：字段命名策略（wire/domain 两层类型）

**触发时机**：Issue #4 (stream-parser) 开工前，Claude Code 抓真实 CC 样本对照架构文档 §4.2 的 TypeScript interface，发现实际字段是 `session_id` snake_case，文档示例是 `sessionId` camelCase。

**Claude Code 提的三方案**：
- A：保留 camelCase 接口 + Parser 字段映射
- B：Schema 改 mixed case（和 CC 原生对齐）
- C：zod transform 自动 snake→camel

**总工拍板**：B' —— 两层类型 + Parser 显式映射
- `packages/shared/src/types/wire/` 保留 CC 原始 snake_case + zod schema
- `packages/shared/src/types/`（domain 层）camelCase
- Parser 的 `normalize` 函数做 wire → domain 显式映射

**为什么不是 CC 倾向的 B**：B 会让 snake_case 一路污染到 React 组件、Zustand store、SQLite schema、skill manifest，整个 codebase 维护成本上升。两层类型保持 boundary 保真 + 内部统一。

**事后评价**：决策正确。两层类型在 Issue #4 实施时无 friction，wire schema 用 `.passthrough()` 兜底未知字段、domain 类型手写 camelCase，新增字段时漏掉会被 TypeScript 编译错误兜底。

### D6：CC 事件流的扁平化

**触发时机**：Issue #4 抓真实 CC 样本时发现 `tool_use`/`tool_result` 在 Anthropic API 规范里是 content block 而不是 top-level event，与架构文档 §4.2 不符。

**Claude Code 提的三方案**：
- X：扁平化为独立事件（每个 content block 一个 domain 事件）
- Y：保真 API message（domain CCEvent 保留 message.content[] 嵌套）
- Z：混合（既发 message-level 又发 block-level）

**总工拍板**：X，加修正 D6a。
- 字段 `isLast` + `messageMeta` 挂在同一 msgId 的最后一个 domain 事件上
- 不发独立的 `message_meta` 事件
- 同一 wire message 产生的所有 domain 事件共享 msgId

**D6 后续**：Claude Code 在实测中发现 CC 2.1.119 实际是**每个 content block 发一个独立 wire assistant event**，与"一个 wire event 带 N 个 blocks"的假设不符。这导致 D6a 的"最后一个事件挂 messageMeta" 退化为"per wire event last"。

**总工二次拍板**：保留 stateless parser，澄清 isLast/messageMeta 语义为"per wire event 快照"，UI 通过 `tool_result` / `result` / 下一个 user turn 来判断"消息真的说完"。

**事后评价**：决策正确但**架构文档撰写时假设错了**。我（总工）在写 §4.2 时假设 stream-json 事件结构和 Anthropic Messages API 结构一致，没对真实 CC 输出校验。这是**M1 撰写架构文档时必须避免的失误**——先抓样本，再写文档。

**遗留**：D6 follow-up（JSDoc 澄清 + README Consumer guide）作为 TD-D6 进入技术债，M1 早期处理。

### D9：CI 跨平台失败修复

**触发时机**：Issue #9 (CI/CD) 实施时三平台 CI 全失败，Vitest 4 的 rolldown native binding 缺失（ubuntu+macos）+ Windows CRLF 违反 .editorconfig（windows）。

**Claude Code 提的两方案**（每个问题独立）：
- 问题 1 方案 A：根 package.json 加 `pnpm.supportedArchitectures` 枚举所有平台
- 问题 1 方案 B：CI 改 `--no-frozen-lockfile`
- 问题 2 方案 A：加 `.gitattributes` 强制 LF
- 问题 2 方案 B：CI workflow 前加 `git config core.autocrlf false`

**总工拍板**：两个问题都走方案 A。

**关键边界判断**：D9 触发了**判断标准的精细化**。Claude Code 在"灰色地带主动停下来问"是对的，但停下来的成本是"我每次都要等"。总工给出新的分类标准：

**可以自主决策**：
- CI workflow 文件（`.github/workflows/*.yml`）的调整
- `package.json` 的 scripts 字段
- 单 package 内部的 tsconfig / vitest.config
- 行业标准的 lint/format 配置
- 工具链已知问题的标准 workaround

**必须停下来问**：
- 根 lockfile 实际 diff（`git diff pnpm-lock.yaml`，不是改 package.json 就触发）
- `.gitattributes` / `.gitignore` 根文件的新增或重大改动
- workspace 级依赖的 add/remove/version bump
- 影响 "git clone 后第一次跑命令的体验" 的改动
- 涉及签名、发布、跨平台打包的配置

**判断原则**：改动是否扩散到仓库之外（未来贡献者、CI 运行时、用户机器）。扩散 = 问。只影响 CI 内部或单 package = 自主决。

**事后评价**：D9 是 M0 期间最有价值的"协作模式校准"。它让后续决策的频率明显下降——M0 后半段 Issue #6/#7/#8 期间，Claude Code 没再因为类似问题停顿。

---

## 四、文档勘误记录

M0 期间共发现并修复 2 次架构文档勘误，**第三次勘误待补**：

### 勘误 1：§4.2 字段命名（已合）

**触发**：D1 决策。  
**内容**：在 §4.2 节标题下加 NOTE，说明示例字段为 camelCase 但实际 CC 为 snake_case，指向两层类型实现。  
**Commit**：`docs(architecture): add note on CC wire vs domain field naming`

### 勘误 2：§4.2 CCEvent 类型定义（已合）

**触发**：D6 决策。  
**内容**：完整重写 §4.2 的 CCEvent discriminated union，从原本的 7 变体改为扁平化方案：
- `assistant_text` / `assistant_thinking` / `tool_use` / `tool_result` 四个独立 domain 变体
- 共享 `MessageEventBase`：msgId / sessionId / uuid / seq / isLast / messageMeta
- thinking block 的 signature 字段保留

**Commit**：`docs(architecture): update §4.2 CCEvent type for flatten decision (D6)` + `docs(architecture): fix markdown fence nesting in §4.2 (re D6)`

### 勘误 3：§4.2 isLast/messageMeta 语义（待补 ⚠️）

**触发**：D6 后续实测。CC 2.1.119 实际是每个 content block 一个独立 wire event，导致 isLast 语义从"per message last"退化为"per wire event last"。

**应补内容**：
- isLast 字段的 JSDoc 说明
- messageMeta 字段的"per wire event 快照"说明
- 新增"Consumer guide：如何判断 message 真的说完"小节，列举 tool_result / result / 下一个 user turn 三种锚点

**优先级**：M1 早期，和 TD-D6 一起做。

---

## 五、技术债清单（M1 早期处理）

| 编号 | 描述 | 影响 | 优先级 |
|------|------|------|--------|
| **TD-D6** | D6 follow-up：StreamParser isLast/messageMeta 语义澄清（JSDoc + README Consumer guide）+ 文档勘误 3 | 文档/注释级，不影响 runtime | M1 第一周 |
| **TD-001** | Playwright Chromium 打包决策（bundle vs 运行时下载，体积 ~150MB） | 应用体积、首次用户体验 | M1 开工前 |
| **TD-002** | preload bundle 138KB 偏大，需要拆 IpcChannels | 启动性能（不严重） | M1 中期 |

---

## 六、M0 提前交付的 M1 功能

Claude Code 在 Issue #7 (IPC) 和 Issue #8 (Hello World) 实施时**精准守住 M0 边界**，没有偷跑 M1 功能。具体自查：

| 项目 | M0 边界 | 实际实现 |
|------|---------|----------|
| assistant_text 渲染 | 纯文本气泡 | 纯文本气泡（emoji 是字符不是 markdown）✅ |
| thinking 渲染 | 等宽朴素文本 | 等宽朴素文本 + 折叠交互（folding 在 02 切片里有） ✅ |
| tool_use/tool_result 渲染 | 简单 JSON 卡片 | 简单 JSON 卡片 ✅ |
| result 渲染 | 简单完成标记 | 绿色背景 + duration + cost，无动画 ✅ |
| markdown 渲染 | M1 范围 | 未实现 ✅ |
| 语法高亮 | M1 范围 | 未实现 ✅ |
| 流式动画 | M1 范围 | 未实现 ✅ |
| 历史会话列表 | M1 范围 | 未实现 ✅ |

**唯一一处 nice-to-have**：右上角状态栏 `v2.1.119 · T***@gmx.es（订阅）✓ 完成`。这对应 Issue #7 的 cc:status channel + 02-mvp-slicing.md 要求的"邮箱脱敏"，**在 M0 范围内**。

**结论**：M0 没有偷跑 M1，M1 起草时所有 02-mvp-slicing.md 的 M1 功能都需要从零实现，不要基于"M0 已经做了一半"的假设缩减 M1 范围。

---

## 七、协作模式观察

M0 验证了**总工 + 发起人 + Claude Code（唯一施工单位）**三角分工的有效性。具体观察：

### 总工（Claude Opus 4.7）的角色实际形态

不是"全程参与施工"，而是：
- **设计阶段**：写 01/02/03/04 四份基础文档，确立架构和方向
- **施工阶段**：在发起人召唤时为决策（D1/D6/D9）提供拍板支持，对 PR 摘要做技术审查
- **不直接接触代码**：不读 Claude Code 写的源文件、不改 git 历史

这种"挂在发起人身后"的角色定位省心、有效。M1 应该继续这套。

### 发起人（yrjm）的角色实际形态

- **决策中枢**：所有架构级问题最终签字
- **技术审查**：本地端到端验收（验收 7 点是关键质量保障）
- **GitHub 上的关键手工操作**：建 repo、关 milestone、打 tag

观察：**发起人的人在场是项目质量的最后防线**。Claude Code 在 PR #14 的 CI 修复后展现的判断力虽然好，但仍有"灰色地带需要拍板"的高频时刻。

### Claude Code 的角色实际形态

- **施工**：写代码、改代码、跑测试、推 PR、自合
- **决策提案**：抛 D1/D6/D9 这种带方案对比的 decision request
- **文档反馈**：发现架构文档不一致时停下来标出来
- **memory 沉淀**：判断标准的细化、术语澄清等都进了它的 memory

观察：**Claude Code 处理"实测发现的真实问题"的能力远超预期**。它不是死板按文档施工，而是会主动校验文档假设并报告偏差。这是 M0 速度快的根本原因。

### 节奏反思

M0 期间一个观察是：**Claude Code 的 PR 节奏从"一个 PR 一个 issue"自然形成，没有刻意要求**。这比"PR 打包多个 issue"更健康——回滚半径小、PR review 颗粒度好。M1 保持这个节奏。

---

## 八、给 M1 起草的输入

M1 issue 清单将由独立 claude.ai 对话中的总工起草。该总工不参与 M0 施工过程，需要本回顾文档作为输入。**关键提示**：

### 1. 撰写 M1 issue 清单前必须做的事

**抓真实 CC + MCP 的端到端样本，再写 issue**。M0 期间 §4.2 的两次勘误就是因为架构文档先写、样本后抓导致的。M1 涉及：

- skill runtime（需要看 CC 接收 system prompt 时的真实行为）
- MCP server（需要看 CC 调用 MCP tool 时的 wire 协议）
- Risk Gate（需要看 CC 工具调用前后的事件流）
- browser tools（需要看 Playwright 实际能拿到什么）

每个模块在写 issue 前抓 1-2 个样本对架构文档 §4.3-4.5 校验，避免 §4.2 的勘误重演。

### 2. 引用 M0 已建立的协议

M1 不要重新发明这些已经稳定的东西：

- **wire/domain 两层类型**（D1）
- **扁平化事件流 + isLast 语义**（D6）
- **判断标准精细化**（D9-1）
- **PR 节奏：一个 PR 一个 issue**
- **PR merge 流程**：CI 绿后自合
- **commit message**：Conventional Commits
- **代码注释中文 + log 英文**（kickoff prompt 已建立）

### 3. M1 issue 数量参考

02-mvp-slicing.md 原计划 M1 是 12 个 issue。基于 M0 实际经验：

- M0 有 8 个 issue + 3 个隐性大决策（D1/D6/D9）
- M1 模块复杂度更高，预计 **12 个 issue + 至少 5-8 个决策点**
- 不要把 M1 issue 切得太细——避免 issue 编号膨胀但实际工作量不足
- 不要把 M1 issue 切得太粗——避免一个 PR 涵盖太多模块

### 4. 必须在 M1 第一周做完的技术债

- TD-D6（含勘误 3）：是后续 M1 模块的语义基础
- TD-001 Playwright Chromium：影响应用打包路径

### 5. M1 不要做的事

- 不要重写 M0 的 5 大模块（除非有 ADR-级的破坏性变更）
- 不要在 M1 引入新的技术栈（Tauri / Vue / 国产模型 BYOK 等都是 v1.5 以后的事）
- 不要做 v1 不做列表里的功能（参见 02-mvp-slicing.md §五）

---

## 九、致谢与里程碑标记

M0 是 OpenTrad 项目第一次真正的"产出 → 验收 → 收官"完整循环。从 2026-04-24 开工到 2026-04-24 收官，所有参与方（项目发起人、Claude Code、总工）的协作模式得到验证。

下一次回顾在 M1 收官时撰写。

**M0 关键数字**：
- 8 issues closed
- 8 PRs merged  
- 3 architecture decisions (D1/D6/D9)
- 3 documentation corrections (2 done, 1 pending)
- 3 technical debts logged (TD-D6, TD-001, TD-002)
- 0 production bugs (M0 不发布生产版本)
- 1 working Hello World demo: `4667ms · $0.009503`

里程碑标签：[`m0-complete`](https://github.com/open-trad/opentrad/releases/tag/m0-complete) (`8748590112bf2f567407f45be438ade77244fd7`)

---

**正式标记 M0 收官，M1 启动权交予新 claude.ai 对话中的总工。**
