# OpenTrad M1 Issue 清单

**文档版本**:1.0
**撰写日期**:2026-04-24
**作者**:总工(Claude Opus 4.7,M1 起草轮次)
**依赖**:`01-positioning.md` v1.1、`02-mvp-slicing.md` v1.0、`03-architecture.md` v1.0、`retrospective-m0.md` v1.0

---

## 本文档的用途

M0 已收官([`m0-complete`](https://github.com/open-trad/opentrad/releases/tag/m0-complete)),骨架可跑、Hello World 端到端通过。M1 目标是把这条最小通路装上"外贸商家能用的外壳"——5 块新模块叠上去,产出 v0.1 preview 可装机版本。

本文档是 M1 的施工清单,与 04-kickoff-checklist §三同款颗粒度。13 个 issue,Claude Code 用 `gh issue create` 批量创建后按依赖顺序施工。

**沿用 M0 协作模式**:一 PR 一 issue、CI 绿后自合、Conventional Commits、判断标准精细化(retrospective §七 D9-1)、代码注释中文 + log 英文。M1 不发明新的协作模式。

---

## §零 引言

### M0 / M1 边界

**M0 完成**:把"应用骨架 + Claude Code 子进程 + stream-json 解析渲染"这条最小数据通路打通,跑通 Hello World demo,验证三件套设计可施工。8 个 issue close、CI 三平台守门、Hello World `4667ms · $0.009503`。

**M1 要做**:在这条数据通路上叠 5 块新模块,让外贸商家能在本地一键装好、跑完 1 个完整 skill,产出 v0.1 preview 可装机版本。

5 块新模块:
1. Skill runtime(SkillLoader + PromptComposer)
2. 内置 MCP server + IPC bridge(独立进程 + Unix socket / named pipe)
3. Browser tools(Playwright + bundled Chromium)
4. Risk Gate(工具级 + 业务级 + 审计日志)
5. 安装/登录引导(PTY + node-pty + xterm.js)

外加 1 个 SQLite 持久化层(M0 完全没碰),1 个 trade-email-writer skill 端到端,1 个 electron-builder 三平台打包。

### M1 不做的事

明确从 M1 范围排除:
- 其他 4 个 P0 skill(supplier-rfq-draft / product-sourcing-1688 / listing-localizer-seo / hs-code-compliance-check)——M2 范围
- 第三方 skill 导入 / skill 市场 UI(F3.4 / F3.5)——M2/M3 范围
- 第三方 MCP server 管理(F4.3)——M2 范围
- 多 session 并发(F1.6)——M2 范围
- 自定义风险规则编辑器(F5.4)——M2/M3 范围
- 公开 v0.1 release(release notes / 安装文档 / FAQ)——M1 收官后单独动作
- macOS / Windows code signing——M3 范围

### 复述 M0 已建立的协议(M1 不重建)

- **wire/domain 两层类型**(D1):wire 层 snake_case + zod schema,domain 层 camelCase,parser 显式映射
- **扁平化事件流 + isLast/messageMeta per-wire-event 语义**(D6 + 勘误 3,见 issue #1)
- **判断标准精细化**(D9-1):改动是否扩散到仓库之外是停下来问的判断锚点
- **代码注释中文 + log 英文**(M0 kickoff prompt)
- **PR 节奏**:一个 PR 一个 issue,CI 绿后自合,issue 描述里写"done"再 close

---

## §一 开工前决策(已拍板,不再讨论)

以下 4 项在 M1 issue 起草前由发起人和总工讨论后确定,作为施工的前置约束:

### D-pre-1:Playwright Chromium 打包(TD-001 落地)

**决策**:**方案 A,bundle Chromium**。electron-builder 打包时通过 `extraResources` 将 Playwright Chromium 二进制内置到 .app/.exe/.AppImage,首次启动校验 bundled 路径优先。

**理由**:
- 外贸商家用户群对 300MB→450MB 的应用体积差异不敏感(对照 Photoshop、WPS 体量)
- 但对"装完了打开还要再下东西"敏感,运行时下载方案体验断点过明显
- ADR-007 已写"打包时 bundle 进去,用户零等待"

**代价**:
- electron-builder 打包时间增加 2-5 分钟,CI release 流水线变慢(开发 CI 不打包,只 lint/typecheck/test,不影响日常)
- 应用体积 ~450MB(三平台 .dmg / .exe / .AppImage)

**回滚条件**:M2 用户反馈强烈或社区贡献者提出"接管已装 Chrome via CDP"的 PR 时,加可选路径(参考 ADR-007 v2 考虑)。

**落地点**:issue #10。

### D-pre-2:M1 = 打包通过 + artifact 可下载,不公开 v0.1 release

**决策**:M1 收官标准是 electron-builder 三平台打包 artifact 在 GitHub Actions 产物里可下载、装机能跑完 trade-email-writer 端到端,**不要求公开 v0.1 release**。

**理由**:
- 02 文档把 v0.1 公开发布压在 M1 偏乐观——release notes / 安装教程 / known issues / 隐私声明 / bug 上报指引是半个文档冲刺
- 退一步装机几天发现问题,质量更稳;直接发可能 release notes 与实际行为对不上
- M0 retrospective lesson:文档先写后校会出错

**落地点**:issue #13 验收只要求"artifact 可下载 + 三平台装机跑通",不要求"GitHub Releases 页面公开"。M1 收官后单独动作做公开发布(归入 retrospective-m1 之后的小动作或 M2 开工前)。

### D-pre-3:skills 目录 M1 范围只做 trade-email-writer

**决策**:M1 只创建 `skills/trade-email-writer/` 完整内容(skill.yml + prompt.md + README + icon + examples),其他 4 个 P0 skill 目录**整个不创建**,留给 M2。

**理由**:
- 占位 manifest 增加维护成本,M1 收益不大
- SkillPicker UI 设计成"已启用 skill"列表风格,M2 加入更多时无需 UI 改动
- 02 文档 §一明确"M1 至少 1 个 skill 完整可用"——只做 1 个就够

**落地点**:issue #7(SkillPicker UI 不显示未做的 skill)、issue #13(只跑 trade-email-writer 端到端)。

### D-pre-4:PTY fallback 从 P1 提到 P0

**决策**:02-mvp-slicing F1.4 标记为 P1,M1 修订为 **P0**,优先级排在安装向导和登录引导之前。

**理由**:CC 的 install 脚本和 `claude auth login` 都是交互式 terminal 流程,非 PTY 模式看不到 https URL,用户无法完成浏览器登录。这是 02 起草时漏算的依赖关系。

**落地点**:issue #3(独立 issue 实现 PTY 基础)→ issue #4 / #5 依赖。

---

## §二 Issue 清单

13 个 issue,按依赖顺序排列。Claude Code 创建时 GitHub issue 编号会接续 M0 编号(M0 占用 #1-#9,M1 预计 #10-#22)。

下文所有 issue 编号均为 **M1 内部序号**,不是 GitHub issue 编号。

### Issue #1:TD-D6 + 勘误 3 + M1 前置样本采集

**标签**:`milestone:M1`、`type:tech-debt`、`P0`、`module:stream-parser`、`module:shared`

**依赖**:无(M0 已交付的 packages/stream-parser 直接接续)

**描述**:

M0 期间 D6 决策"扁平化事件流"实施时发现 CC 2.1.119 实际是每个 content block 一个独立 wire assistant event,导致 isLast/messageMeta 语义从原文档的"per message last"退化为"per wire event last"(详见 retrospective §三 D6 后续 + §四勘误 3)。

补三件事:
1. **代码侧**:packages/stream-parser/src/types.ts 上的 isLast / messageMeta 字段加 JSDoc,说明 per-wire-event 语义
2. **README 侧**:packages/stream-parser/README.md 增加"Consumer guide"小节,列举 tool_result / result / 下一个 user turn 三种"消息真说完"的锚点 + 代码示例
3. **架构文档侧**:在 docs repo 提 PR 修订 03-architecture.md §4.2,在已有 NOTE 之后新增"isLast/messageMeta 语义"小节(15-30 行)

**同时本 issue 内嵌前置子任务:抓真实样本**(retrospective §八.1 要求,为后续 issue #8 / #11 准备):
- 样本 (a):用 fs-mcp 或 echo MCP 让 CC 调用一次工具,记录完整 stream-json 输出
- 样本 (b):一次 user-confirmation-required 的 tool_use → tool_result 序列(可用 CC 自身的 Bash tool 触发权限提示模拟)

样本整理成 redacted markdown 提交到 `open-trad/docs` repo `evidence/m1-prep-samples/` 目录,模式参考调研期 cc-local-cli-probe.redacted.md。

**要做**:
- 给 packages/stream-parser/src/types.ts 的 `MessageEventBase.isLast` / `MessageEventBase.messageMeta` 字段加完整 JSDoc(注明:每个 wire event 独立判断,不是整条 message;同 msgId 的下一个 event 到达前,本 event 的 isLast 不可信)
- 在 packages/stream-parser/README.md 新增 `## Consumer guide: 如何判断 message 真的说完` 小节,给出 3 个锚点示例(tool_result 出现 / `result` 事件 / 下一个 user turn),每个锚点附简短 TS 代码片段
- 在 docs repo 提 PR:`docs(architecture): add isLast/messageMeta semantics section (errata-3, TD-D6)`,§4.2 末尾追加小节
- 抓 2 份样本提交到 docs repo `evidence/m1-prep-samples/01-mcp-tool-wire.md` 和 `02-tool-confirmation-flow.md`,每份内含完整 NDJSON 截选 + 字段注释 + 抓样本时的 CC 版本号
- M0 现有 49 个单测继续全绿,如有新 case 一并加上

**验收**:
- StreamParser 公开 API 上 isLast / messageMeta 有 JSDoc 完整说明 per-wire-event 语义
- README Consumer guide 至少给出 3 个锚点示例代码
- docs repo 03-architecture.md §4.2 末尾新增"isLast 语义"小节(已合)
- evidence/ 目录有 ≥2 份样本 .md
- `pnpm --filter @opentrad/stream-parser test` 全绿

**预估**:1-1.5 天

---

### Issue #2:SQLite 持久化层

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:desktop`

**依赖**:无(可与 #1 并行)

**描述**:

按 03-architecture.md §五落地 SQLite 数据库,作为后续 history / risk_rules / audit_log / settings / installed_skills 等模块的存储基础。M0 完全没碰 SQLite,这是 M1 第一个有用户可见持久状态的模块。

按 ADR-004,直接用 better-sqlite3 同步 API,**不引入 ORM**;用 zod 做 row 运行时校验。

**要做**:
- 添加 `better-sqlite3` 和 `zod`(若 #1 未引入)依赖到 `apps/desktop`
- 创建 6 张表(沿用 03 §五 schema):`sessions` / `events` / `risk_rules` / `audit_log` / `settings` / `installed_skills`,SQL 字符串放 `apps/desktop/src/main/services/db/schema.ts`
- 数据库初始化:`apps/desktop/src/main/services/db/init.ts`,首次启动建表 + 写入 `schema_version=1` 到 settings(预留迁移机制,M1 不实现迁移)
- 数据库文件路径:macOS/Linux `~/.opentrad/opentrad.db`,Windows `%APPDATA%\OpenTrad\opentrad.db`,目录不存在则建
- service 层封装:
  - `SessionService`:create / list(分页 limit/offset)/ get / updateStatus / delete / cascadeDelete
  - `SettingsService`:get / set(JSON 序列化 + zod 校验)/ reset
  - `InstalledSkillService`:list / install / enable / disable
  - `EventService`:append / readBySession(支持事件回放,issue #12 用)
  - `RiskRuleService`:findMatching / save / list / delete(为 issue #11 准备)
  - `AuditLogService`:append / queryBySession / queryByDateRange(为 issue #11 准备)
- IPC 暴露:`session:list` / `session:get` / `session:delete` / `settings:get` / `settings:set` / `installed-skill:list`
- 互斥保护:启动时尝试创建 `~/.opentrad/.lock` 文件(包含 PID),已存在则提示"OpenTrad 已在另一个窗口运行"并退出

**架构决策**:
- shared types 放 `@opentrad/shared`(沿用 D1 wire/domain 两层规范)
- 不做 SQLite WAL 模式(单写者足够)

**验收**:
- `pnpm dev` 启动后 `~/.opentrad/opentrad.db` 生成,`sqlite3 .schema` 6 张表 + 索引完整
- 写读循环:renderer 通过 IPC 写一个 setting,重启后能读回
- 写 ≥10 条 sessions,`session:list { limit: 5, offset: 0 }` 按 `updated_at DESC` 返回 5 条,`offset: 5` 返回剩余
- 单元测试覆盖 session-service / settings-service / installed-skill-service 主路径,fixture 用 in-memory sqlite
- 应用退出时 db 干净关闭,无 `-journal` 残留
- 同时启动两个 OpenTrad 窗口时第二个被互斥保护拒绝

**预估**:2-3 天

---

### Issue #3:PTY fallback + xterm.js 集成

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:desktop`

**依赖**:无(可与 #1 / #2 并行)

**描述**:

按 02 F1.4(M1 提到 P0,见 D-pre-4)实现 PTY 模式,使用 `node-pty` + `xterm.js`。后续 issue #4(安装向导)和 #5(登录引导)都依赖它——CC 的 install 脚本和 `claude auth login` 都是交互式 terminal 流程,非 PTY 模式看不到 URL。

注意 `node-pty` 的预编译 binary 在三平台兼容性(M0 D9 时遇到过 rolldown native binding 问题,这次同类风险),需要在 `ci.yml` 三平台都验证 build。

**要做**:
- 添加 `node-pty` + `@xterm/xterm` + `@xterm/addon-fit` + `@xterm/addon-web-links` 依赖
- `apps/desktop/src/main/services/pty-manager.ts`:封装 node-pty,管理 PTY session 生命周期(spawn / write / resize / kill / cleanup)
- IPC channel:
  - `pty:spawn { command, args, cwd, env? }` → renderer 发起,主进程 spawn 后返回 ptyId
  - `pty:write { ptyId, data }` → renderer 写键盘输入
  - `pty:resize { ptyId, cols, rows }` → renderer 同步窗口大小
  - `pty:kill { ptyId }` → 主动终止
  - `pty:data { ptyId, data }` → 主进程 → renderer 推送 stdout/stderr
  - `pty:exit { ptyId, code }` → 主进程 → renderer 通知退出
- `apps/desktop/src/renderer/components/ui/TerminalPane.tsx`:xterm.js 包装,提供 React 组件
- TerminalPane 默认折叠在侧边栏底部("打开 terminal" / "关闭"按钮),用户主动展开
- 跨平台 shell 选择:macOS `zsh`、Linux `bash`、Windows `pwsh.exe`(回退 `cmd.exe`)
- 应用退出时清理所有 PTY 进程

**要确认的事项(决策点 D-M1-1)**:
- node-pty 在 Electron 主进程 require 时需要 `electron-rebuild`(对应 prebuild),build 流程要加这一步。Claude Code 实施时如遇到 native module 问题,沿用 D9-1 判断标准:CI workflow 调整可自主决策

**验收**:
- 在 macOS / Windows / Linux 三平台都能打开 TerminalPane 跑 `echo hi` 看到输出
- 输入键盘命令(包括方向键、Ctrl-C、Tab 补全)PTY 行为正确
- 调整窗口宽度 PTY columns / rows 自动 resize
- TerminalPane 关闭后 pty 进程清理干净(`ps aux | grep` 验证)
- CI 三平台全绿,`pnpm test` 包含 pty-manager 基础单测(spawn echo 后 read stdout)
- M0 既有所有测试和 typecheck 全绿

**预估**:1.5-2 天

---

### Issue #4:CC 安装向导 + 检测强化

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:cc-adapter`、`module:desktop`

**依赖**:#2、#3

**描述**:

按 02 F6.1 + F1.1 增强,实现首次启动的安装引导流程。M0 的 cc-adapter `detectInstallation` 已能判断"装没装",但没做"如何装"。流程对应 03 §7.1 的 OnboardingStep1。

**要做**:
- `apps/desktop/src/main/services/installer.ts`:封装 CC 安装命令(平台分支)
  - macOS / Linux:`curl -fsSL https://claude.ai/install.sh | bash`
  - Windows:对应 PowerShell 命令(参考 Anthropic 官方文档,如不可用则提示"请访问 claude.ai 手动安装")
- `apps/desktop/src/renderer/features/onboarding/InstallStep.tsx`:UI 组件
- IPC channel:
  - `installer:run-cc-install` → 主进程通过 PTY(issue #3)开 terminal 跑安装命令,渲染层观看
  - `cc:detect-loop { intervalMs }` / `cc:detect-loop-stop` → 主进程定时轮询 `detectInstallation()`,通过 `cc:status` 推送
- onboarding 路由 + `OnboardingGate` 组件(检查 `settings.onboarded` 标志,未 onboarded 时强制进入 onboarding)
- "我已经自己装好了"按钮(跳过自动安装,直接重新检测)
- "跳过引导"按钮(进入主界面但保持 onboarded=false,下次启动再问)
- UI 醒目提示:"安装命令来自 claude.ai 官方,你可以读完命令再决定是否执行"(隐私声明)
- **不写凭证文件、不读 ~/.claude**(SECURITY.md 红线)

**架构决策**:
- 自动安装失败时(网络问题、权限拒绝),InstallStep 显示原始 PTY 输出 + 链接到 docs.claude.com 手动安装文档,**不自动重试**(避免连续失败)
- detect-loop 间隔 3 秒,5 分钟无 installed=true 自动停止并提示用户

**验收**:
- 在没装 CC 的全新机器(可用 Docker Ubuntu 镜像或新 macOS VM 模拟)上,首次开 OpenTrad 自动跳到 `/onboarding/step1`
- 点"一键安装",TerminalPane 里看到完整 install 脚本输出,装完 `claude --version` 在主进程能拿到
- 装完后 5 秒内自动进入 step 2(login,issue #5)
- 在已装 CC 的机器上,首次开应用 step 1 一闪而过,直接进 step 2
- 跨平台:三平台的安装命令各自验证一遍(Windows 若不支持 install.sh,验证手动安装提示路径)
- "跳过引导"测试:点击后进入主界面,SkillPicker 显示但不可用,提示"请先完成 onboarding"

**预估**:1.5-2 天

---

### Issue #5:登录引导 + auth status 完整化

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:cc-adapter`、`module:desktop`

**依赖**:#4

**描述**:

按 02 F1.3 + F6.2 + 03 §7.1 OnboardingStep2,完整化 CC 登录状态识别 + 引导用户登录。

M0 已实现状态栏邮箱脱敏(retrospective §六 `v2.1.119 · T***@gmx.es(订阅)✓ 完成`),即 `getAuthStatus` 已能 parse `claude auth status --text` 的输出。M1 把 auth 状态从"展示"扩展到"未登录时引导用户登录"。

**要做**:
- 增强 `cc-adapter/src/cc-manager.ts` 的 `getAuthStatus`:
  - 完整识别三种状态:未登录 / Subscription / API key
  - 邮箱脱敏:`u***@example.com` 形式(不存原始邮箱,UI 仅展示脱敏后)
- `apps/desktop/src/renderer/features/onboarding/LoginStep.tsx`
- IPC channel:`auth:login-flow`(主进程开 PTY 跑 `claude auth login --claudeai`,推送 PTY 数据到 renderer + 周期性轮询 `getAuthStatus`)
- URL 提取逻辑:从 PTY stdout 用 regex 抽 `https://claude.ai/...` 链接,渲染层展示"点击打开浏览器登录"按钮(主进程用 `shell.openExternal` 打开)
- 登录成功后 PTY 自动关闭,进入 step 3
- API key 模式:Login UI 提供"使用 API key 登录"备选路径,运行 `claude auth login --apiKey ...` 流程
- 主界面状态栏(M0 已有的)继续用,但要更新为"已登录 u***@xxx.com (Claude Pro)"格式,无登录时显示"未登录,点击登录"
- **绝对不读 `~/.claude/.credentials.json`**(SECURITY.md 红线 + retrospective 反复强调)

**架构决策**:
- 登录失败时(网络、auth 服务器问题),LoginStep 显示完整 PTY 输出 + 链接到 docs.claude.com 故障排查文档
- 登录中支持"取消"按钮 → kill PTY 进程 + 回退到 LoginStep 初始状态

**验收**:
- 未登录状态下点"登录 Claude",TerminalPane 显示完整登录提示,URL 被自动高亮且"打开浏览器"按钮工作
- 完成浏览器登录后,UI 在 5 秒内自动检测到 loggedIn=true,跳转 step 3
- 已登录状态下重启 OpenTrad,主界面状态栏显示账号(脱敏后),onboarding step 1 + 2 都被跳过
- API key 模式登录(`claude auth login --apiKey ...`)能识别 `method: 'api_key'` 并展示
- 任何阶段都不访问 `~/.claude/` 下的文件(grep 代码确认)

**预估**:1.5 天

---

### Issue #6:Skill runtime 后端

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:skill-runtime`、`module:shared`

**依赖**:#2

**描述**:

按 03 §4.3 创建 `packages/skill-runtime`,实现 SkillManifest 类型 + SkillLoader + PromptComposer 三件套。

**要做**:
- 新建 `packages/skill-runtime`,沿用 monorepo 配置(参考 M0 packages/* 模板)
- **SkillManifest 类型放 `@opentrad/shared`**(决策:多个包要 import,统一中心管理),package skill-runtime 只 re-export
- `@opentrad/shared/src/types/skill.ts`:`SkillManifest` / `SkillInput` 类型(从 03 §4.3 抄)
- `@opentrad/shared/src/schemas/skill.ts`:zod schema(运行时校验 skill.yml 内容)
- `packages/skill-runtime/src/loader.ts`:`SkillLoader`
  - `loadFromDirectory(dir)`:读 dir/skill.yml(用 `js-yaml`)+ dir/prompt.md,组装 SkillManifest,过 zod 校验
  - `loadBuiltinSkills(skillsDir)`:扫描 skills/ 目录,加载所有 manifest
  - `loadUserSkills(userSkillsDir)`:从 `~/.opentrad/skills/` 加载用户导入的 skill(M1 不暴露导入 UI,但 loader 接口先就位)
  - 缓存策略:启动时一次性加载,运行期不变;加载失败的 skill 标记 `loadError` 不抛异常(单 skill 错误不阻塞其他)
- `packages/skill-runtime/src/composer.ts`:`PromptComposer`
  - `compose(skill, inputs)`:读 promptTemplate 文件 → mustache-style `{{varName}}` 替换 → 拼通用前缀 → 返回最终 prompt
  - 必填字段缺失时抛 `ValidationError`
  - 通用前缀注入:`You are operating within OpenTrad's <skillId> skill. <skill.description>. Stay within scope.`
- 单元测试:覆盖 loader / composer 主路径,fixture 用一个最简 skill(`__fixtures__/sample-skill/`)

**架构决策(D-M1-2)**:
- SkillLoader 加载失败的 skill 不阻塞其他 skill 加载(per-skill try/catch)
- PromptComposer 不支持嵌套 mustache(M1 简化,M2 视需求加)
- skill manifest 的 `version` 字段 M1 仅校验存在,不做 semver 比对

**验收**:
- `pnpm --filter @opentrad/skill-runtime test` 全绿
- 单测覆盖:loader 加载 fixture skill(manifest 解析正确)、composer 给定 fixture inputs 输出可读 prompt(带前缀 + 替换后的模板)、zod 拒绝不合法 manifest(缺 id / 错误 risk_level / 未识别的 input.type)
- shared 类型可被其他包 import,`@opentrad/shared` 重新暴露完整
- loader 加载 5 个 manifest 中 1 个故意损坏,只该 skill 标记 loadError,其他 4 个正常加载

**预估**:1.5-2 天

---

### Issue #7:Skill UI(SkillPicker + 输入表单)

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:desktop`

**依赖**:#6、#2

**描述**:

按 02 F3.2 后半 + F3.3 后半 + 03 §六 SkillStore 设计,实现 skill 在 UI 侧的展示和触发。

**要做**:
- `apps/desktop/src/renderer/stores/skill.ts`:Zustand SkillStore(loadSkills / enableSkill / disableSkill,数据来自 IPC)
- `apps/desktop/src/renderer/components/SkillPicker.tsx`:左侧栏组件,展示已启用 skill 列表,点击选中
- `apps/desktop/src/renderer/features/skills/SkillInputForm.tsx`:根据 manifest.inputs 动态生成表单
  - 5 种 input 类型:text / textarea / url / select / file
  - 必填字段未填时禁用提交按钮 + 高亮提示
  - file 类型:用 Electron `dialog.showOpenDialog`,读 base64 注入(M1 限制 1 MB 内,超过提示用户)
- `apps/desktop/src/renderer/features/skills/SkillWorkArea.tsx`:中栏容器
  - 选中 skill 后,中栏先显示输入表单
  - 点"发送"后,表单收起、对话流展开(从 issue #9 接通真实 startTask)
- IPC channel:`skill:list` 接 `SkillLoader.loadBuiltinSkills` + `loadUserSkills`,返回所有 manifest

**架构决策(D-pre-3 + D-M1-3)**:
- M1 SkillPicker **只显示 trade-email-writer 一个**(此时该 skill 还未实现,显示占位),不显示未做的 4 个 P0 skill
- skill 输入表单**显示在中栏对话流上方**(同一工作区,选中 skill 后展开表单,提交后表单替换为对话流),不走独立路由 `/skills/:id`

**验收**:
- 启动应用后 SkillPicker 显示 1 个 skill(trade-email-writer placeholder UI)
- 点击 skill,中栏展开输入表单,字段与 manifest 一致(此 issue 用 fixture manifest;真实的 trade-email-writer manifest 在 issue #13 落地)
- 表单缺必填项时提交按钮禁用 + 红框
- 表单提交时调用 sessionStore.startTask(此 issue 暂用 stub:console.log 记录调用),issue #9 真接通

**预估**:1.5 天

---

### Issue #8:MCP server 骨架 + IPC bridge

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:mcp-server`、`module:desktop`

**依赖**:#1(样本就位)、#2(SQLite)

**描述**:

按 03 ADR-006 + §4.4 + §三"进程拓扑",创建 `apps/mcp-server`,作为独立 Node.js 进程被 CC 通过 stdio 拉起。同时实现与 desktop 主进程的 IPC bridge(Unix socket / named pipe + JSON-RPC 2.0)。

**这是 M1 复杂度最高的单一 issue**,包含三件:
1. `apps/mcp-server` 骨架(stdio MCP server,基于 `@modelcontextprotocol/sdk`)
2. desktop 主进程的 IPC bridge server(Unix socket / named pipe)
3. mcp-server 端的 IPC bridge client + 一个 echo tool 端到端跑通

**要做**:
- 新建 `apps/mcp-server`,使用 `@modelcontextprotocol/sdk`
- `apps/mcp-server/src/index.ts`:stdio MCP server 入口,从 env 读 `OPENTRAD_IPC_SOCKET` 和 `OPENTRAD_SESSION_ID`
- `apps/mcp-server/src/ipc-bridge.ts`:JSON-RPC 2.0 client,连 desktop 主进程 socket(retry 3 次,失败后 graceful degrade 见下)
- `apps/desktop/src/main/services/ipc-bridge-server.ts`:JSON-RPC 2.0 socket server
  - macOS / Linux:Unix domain socket `~/.opentrad/ipc.sock`(应用启动时创建,退出清理)
  - Windows:named pipe `\\.\pipe\opentrad-ipc`
  - 跨平台封装在同一接口,内部分支
- 实现 4 个 JSON-RPC 方法(M1 mock + 真实分阶段):
  - `risk-gate.request`:此 issue 用 mock 返回 `{ decision: 'allow' }`,真实在 issue #11
  - `audit.log`:此 issue 直接写 `audit_log` 表(用 #2 的 AuditLogService)
  - `draft.save`:此 issue 写 `~/.opentrad/drafts/` 文件
  - `session.metadata`:此 issue 从 SessionService 查 + 返回基础元数据
- `apps/mcp-server/src/tools/index.ts`:tool 注册框架(`OpenTradTool` 接口,带 `riskLevel: 'safe' | 'review' | 'blocked'` 和 `category`)
- 添加最简 `echo` tool 验证端到端(safe,不需要 RiskGate 拦截)
- electron-builder.yml 增加 `apps/mcp-server` 作为 extraResources(M1 这步先开个 TODO,真打包在 issue #13 做)

**架构决策(D-M1-4 + D-M1-5)**:
- mcp-server 是 **per-task 拉起**(CC 启动时通过 stdio 拉起),非常驻
- desktop 启动时创建 IPC bridge socket;退出时清理 socket 文件
- IPC bridge 连接失败时(单独跑 mcp-server 测试场景),mcp-server **graceful degrade**:
  - tools 注册仍正常(方便单元测试)
  - `risk-gate.request` 默认返回 `deny`(安全侧,避免无确认机制下任意调用)
  - `audit.log` 在内存里 buffer,连上后批量发送
  - 后续重连成功后切换回正常模式

**验收**:
- 手动启动 desktop 主进程,`~/.opentrad/ipc.sock` 文件创建
- 单独跑 `apps/mcp-server`(env 设好 socket 路径)能连上 IPC bridge,发一个 `audit.log` RPC 调用,主进程能在 `audit_log` 表里看到记录
- 把 mcp-server 接入真实的 CC session(配 mcp-config 跑 `claude -p ...`),CC `/mcp` 命令能看到 opentrad 的 echo tool 并调用
- 单独跑 mcp-server 不连 desktop 主进程时,echo tool 仍能调用,但 risk-gate.request 拒绝
- 单元测试覆盖 ipc-bridge-server 的 socket 协议层 + mcp-server 的 tool 注册
- Windows named pipe 路径 `\\.\pipe\opentrad-ipc` 能正确建立(三平台 CI 都跑通)

**预估**:3-4 天(M1 最大 issue)

---

### Issue #9:MCP Config Writer + 第一波 safe-only tools

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:desktop`、`module:mcp-server`

**依赖**:#8

**描述**:

按 03 §4.4 实现 `McpConfigWriter`,以及 mcp-server 的第一批 safe-level tools(不含 browser 工具,那是 issue #10 的事)。同时把 M0 的 cc-adapter `startTask` 接到 McpConfigWriter,让 sessionStore.startTask 真正能跑通完整链路。

**要做**:
- `apps/desktop/src/main/services/mcp-writer.ts`:`McpConfigWriter`(按 03 §4.4)
  - `generateForSession(sessionId, manifest)`:写入 `os.tmpdir()/opentrad-{sessionId}.mcp.json`,包含 mcpServers.opentrad 配置(command 用 process.execPath,env 注入 OPENTRAD_IPC_SOCKET / OPENTRAD_SESSION_ID)
  - `cleanup(sessionId)`:任务结束后删除临时文件
- 修改 `cc-adapter/src/cc-manager.ts` 的 `startTask`:接收 `mcpConfigPath` 和 `allowedTools` 参数,在 spawn 时传 `--mcp-config <path> --strict-mcp-config --allowedTools <space-separated>`(参考 M0 hello world 的 spawn 实现扩展)
- 接通 `sessionStore.startTask`(从 issue #7 留下的 stub):
  1. `SkillLoader.load(skillId)` → manifest
  2. `PromptComposer.compose(manifest, inputs)` → fullPrompt
  3. `McpConfigWriter.generateForSession(sessionId, manifest)` → configPath
  4. `CCManager.startTask({ sessionId, prompt: fullPrompt, mcpConfigPath: configPath, allowedTools: manifest.allowedTools })`
  5. 订阅 handle.events,通过 IPC `cc:event` 推 renderer
  6. result 事件到达后,`McpConfigWriter.cleanup(sessionId)` 清临时文件;写 sessions 表 status='completed' + 总 cost
- 实现第一批 safe-only mcp-server tools:
  - `draft_save({ filename, content })`:走 IPC bridge `draft.save` → 写 `~/.opentrad/drafts/{date}-{filename}.md`(safe,直接放行)
  - `hs_code_lookup({ description })`:M1 用 mock 返回固定 3 个 candidate(safe,M2 接真实接口)
  - `session_get_metadata({ sessionId })`:走 IPC bridge `session.metadata`(safe,read-only)
- 这三个 tool 都注册在 mcp-server 的 tools 列表,riskLevel='safe'

**验收**:
- 用户从 SkillPicker 选 trade-email-writer placeholder + 填表 + 点发送(此时用 fixture skill,真实 trade-email-writer 在 issue #13)
- CC 正常拉起 mcp-server,通过 `/mcp` 能看到 4 个 tools(echo + draft_save + hs_code_lookup + session_get_metadata)
- 任务结束后 `ls $TMPDIR/opentrad-*.json` 为空(临时配置已清理)
- draft_save 调用后 `~/.opentrad/drafts/` 出现新文件
- 端到端跑一次"假" trade-email-writer(prompt 让 CC 调用 draft_save 保存一封简短 mock 邮件),观察事件流完整、UI 显示文本气泡 + tool 调用卡片
- sessions 表 status 正确从 'active' → 'completed',total_cost_usd 写入

**预估**:2 天

---

### Issue #10:Browser tools 包 + Playwright + Chromium bundle

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:browser-tools`、`module:mcp-server`

**依赖**:#9

**描述**:

按 03 ADR-007 + D-pre-1(bundle Chromium)创建 `packages/browser-tools`,把 Playwright 的浏览器自动化能力封装暴露给 mcp-server。

落地 TD-001:Chromium 在 electron-builder 打包时通过 extraResources 内置,首次启动校验 bundled 路径优先。

**要做**:
- 新建 `packages/browser-tools`,封装 Playwright 1.5+
- `packages/browser-tools/src/browser-service.ts`:`BrowserService` 类(单例,懒加载)
  - `launch()`:启动 Chromium(优先用 bundled 路径,从 env `PLAYWRIGHT_BROWSERS_PATH` 读;回退 npm 安装的 Playwright Chromium)
  - `newPage()`:打开新 tab,返回 pageId
  - `getPage(pageId)`:返回 Page 实例
  - `close()`:关闭 browser
  - `cleanup()`:应用退出时调用,关闭所有 page + browser
- 实现 3 个 tool 接口(在 `apps/mcp-server/src/tools/browser/`):
  - `browser_open({ url })`:打开 URL,返回 `{ pageId, title }`
  - `browser_read({ pageId })`:返回简化的 DOM snapshot(用 `page.evaluate` 提取 visible text + 关键 elements 摘要,**避免传整个 HTML**,目标 ≤5KB 文本)
  - `browser_screenshot({ pageId })`:返回 base64 PNG(限制 viewport 1280x720,避免巨大图)
- 这三个 tools 注册时 `riskLevel='review'`(需要用户确认),但本 issue 用 mock RiskGate 拦截(都直接 allow),真实弹窗在 issue #11
- electron-builder.yml 配置:
  - `extraResources`:Playwright Chromium 二进制 + 必要依赖打到 `resources/playwright/`
  - 启动时 `BrowserService.launch` 通过 env `PLAYWRIGHT_BROWSERS_PATH` 指向 bundled 路径
- 跨平台验证:三平台都能拉起 Chromium 跑 browser_open

**架构决策**:
- M1 不做"接管用户已装 Chrome"路径(ADR-007 v2 考虑)
- DOM snapshot 提取算法 M1 简化:抓 `<title>`、`<h1>-<h3>`、可见 text(用 `innerText` 限定根元素深度 5)、`<a>` 链接(限 50 个)。M2 视 skill 需求增强

**验收**:
- `pnpm dev` 启动后,从 mcp-server 调用 `browser_open("https://example.com")` 能拉起 Chromium 看到页面
- `browser_read` 返回结构化 DOM 摘要(可读、≤5KB)
- `browser_screenshot` 返回的 base64 能解码成 PNG(在 macOS / Windows / Linux 都验证)
- electron-builder 三平台打包后(本地手动 `pnpm build` 验证,CI 集成留 issue #13),.dmg / .exe / .AppImage 体积在 350-500MB 范围
- 解压打包后的应用,Resources/playwright/ 下有完整 Chromium binary
- **断网验证**:断网情况下打包后的应用能正常拉起 Chromium(说明 bundle 真生效,无运行时下载)

**预估**:2-3 天

---

### Issue #11:Risk Gate 引擎 + UI + 业务级 stop_before

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:risk-gate`、`module:desktop`、`module:mcp-server`

**依赖**:#8、#2、#10

**描述**:

按 03 §4.5 + §7.3 + 02 F5.1 + F5.2 实现完整 RiskGate。

3 个层面:
1. `packages/risk-gate`(策略引擎)
2. mcp-server middleware(每个 tool 调用前拦截)
3. desktop UI(RiskGateDialog 弹窗 + 业务级业务卡片)

**要做**:
- 新建 `packages/risk-gate`
- `packages/risk-gate/src/gate.ts`:`RiskGate` 类(按 03 §4.5)
  - `check(req)`:四步判断
    1. blocked → 直接 deny
    2. safe + 无 businessAction → 直接 allow + audit
    3. matching rule(查 `risk_rules` 表)→ 用规则的 decision
    4. 弹窗请求用户确认(通过 IPC bridge → desktop renderer)
  - `findMatchingRule`:查 RiskRuleService(#2 已实现)
  - `saveRule`:写 RiskRuleService(`allow_always` 时)
  - `auditLog`:写 AuditLogService(#2 已实现),每次 check 都记录(automated allow / deny / user 决策)
  - `promptUser`:通过 IPC bridge `risk-gate.request` 调主进程 → 主进程 IPC `risk-gate:confirm` 推 renderer 弹窗 → 渲染层用户决策 → 反向回传
- mcp-server middleware:
  - `apps/mcp-server/src/middleware/risk-gate.ts`
  - 每个 tool 执行前调用 `RiskGate.check(req)`
  - allow / allow_once / allow_always → 继续执行
  - deny → 返回 MCP error 给 CC,error message 解释原因
- desktop renderer:
  - `apps/desktop/src/renderer/components/RiskGateDialog.tsx`:工具级确认卡片
    - 展示 toolName / params(脱敏)/ skill 名 / category
    - 4 个按钮:[允许一次] [以后都允许] [拒绝] [编辑后再发]
    - 参数脱敏:URL / 邮箱 / 文件路径中的用户名替换为 `<REDACTED>` 显示给用户但传后端时是原值
  - `apps/desktop/src/renderer/components/BusinessActionCard.tsx`:skill manifest 的 `stopBefore` 触发时显示业务级卡片
    - 例:"即将发送 RFQ 给供应商 ShenZhen ABC。收件人:abc@example.com。内容预览:..."
- 业务级 vs 工具级判断逻辑:
  - 业务级触发条件:skill manifest.stopBefore 列表里包含 toolName 或匹配的 businessAction
  - 优先级:业务级 > 工具级(同一 tool 调用如果是业务级 stop_before,展示业务卡片;否则工具卡片)
- /settings/risk 子页:
  - 展示 risk_rules 列表 + 每条规则的"删除"按钮
  - 审计日志查看(简表格,按 timestamp DESC,分页 50/页)

**架构决策(D-M1-6)**:
- "编辑后再发"按钮 v1 简单实现:返回 deny + 在 audit_log 记录原因 `user_requested_edit`,让 CC 重新生成 tool input(等价于 deny + 让 CC 再试一次)。完整的"用户改 params 再 allow"留 v1.5
- 用户拒绝后 CC 收到 MCP error,会自己决定是否重试(M1 不做强制中止)
- audit_log 全量记录:每次 check 调用都记录(包括 automated allow / deny),保证用户可追溯

**验收**:
- 跑 trade-email-writer 调用 draft_save(safe)→ 不弹窗,直接执行,audit_log 有 `automated:true` 记录
- 跑 trade-email-writer 调用 browser_open(review)→ 弹工具级 RiskGateDialog,按"允许一次"通过,audit_log 有 `automated:false` + decision='allow_once' 记录
- 选"以后都允许"后,risk_rules 表新增规则,下次同 skill+tool 调用直接放行(不弹窗,但 audit_log 仍记录 `automated:true` + `rule_matched:true`)
- 在 trade-email-writer manifest.stopBefore 加 `send_email`(用 mock tool 触发)→ 显示业务级 BusinessActionCard
- /settings/risk 页能看到规则列表,删除规则后下次同样调用重新弹窗
- 拒绝时 CC 收到结构化 MCP error,事件流不中断

**预估**:3 天

---

### Issue #12:Markdown 渲染 + 完整 Chat UI + 历史/设置/导出

**标签**:`milestone:M1`、`type:feat`、`P0`、`module:desktop`

**依赖**:#2、#6、#7、#9

**描述**:

M0 的 Hello World 已有"按钮 + 纯文本气泡 + JSON 卡片"的简陋版,M1 升级到完整 Chat UI:

- F2.3 Markdown 渲染(react-markdown + remark-gfm)
- F6.4 完整输入框(多行 / 粘贴图片 / 拖文件 / Enter 发送 + Cmd-Enter 换行)
- F6.5 历史会话列表(读 SQLite sessions 表,左侧栏,点击恢复)
- F6.6 设置面板(/settings,语言 / 主题 / CC 状态展示)
- F6.7 导出(对话 markdown / 表格 CSV)
- 三栏布局完善(MainLayout 按 03 §六组件树)

**这个 issue 偏大**,但内部各部分耦合度高(都是渲染层 + 都依赖 SQLite + 都依赖完整 IPC)。**允许实施时拆 12a / 12b 二次提交**:
- 12a:Markdown + Chat UI 主体 + 三栏布局完善
- 12b:历史 + 设置 + 导出 + i18n

**要做**:
- 添加 `react-markdown` + `remark-gfm` + `react-syntax-highlighter` + `i18next` + `react-i18next` 依赖
- `components/MessageBubble.tsx`:支持 markdown 渲染
  - 代码块带"复制"按钮,语法高亮(用 react-syntax-highlighter)
  - 表格识别,渲染为 HTML 表格
- `components/ToolCallCard.tsx` + `ToolResultCard.tsx`:从 M0 简版升级
  - 参数 JSON 默认折叠,点击展开
  - tool_result 长文本默认折叠到前 200 字符 + "展开"
- `components/InputBox.tsx`:多行 textarea
  - Enter 发送(默认),Shift-Enter 换行
  - 设置项可切换为 Cmd/Ctrl-Enter 发送(F6.4)
  - 拖文件:本地文件转 base64 attach(限 1MB)
  - 粘贴图片:从 clipboard 读 image 直接 attach(给 CC 用 see_image)
- `features/history/HistoryList.tsx`:左侧栏组件
  - 读 IPC `session:list`,展示 title / skill / 时间(相对时间,如"2 小时前")
  - 当前 session 高亮
  - 点击 session 通过 sessionStore.resume 恢复(参考 02 F1.5,但完整 resume in M2;M1 simplified:用 events 表回放渲染所有事件,**不重启 CC 子进程**——M1 只做"查看历史",真正"继续聊天"留 M2)
- `features/settings/SettingsPage.tsx`:三栏 settings 路由
  - `/settings`:概览
  - `/settings/general`:语言(zh-CN / en)、主题(light / dark / system)
  - `/settings/cc`:CC 版本、登录状态、安装路径(只读 + "重新检测"按钮)
  - `/settings/risk`:见 issue #11
  - `/settings/about`:版本、GitHub 链接、AGPL-3.0 声明
- `features/export/ExportButton.tsx`:
  - 对话导出 .md(使用 markdown 序列化所有消息)
  - 表格识别后单独"导出 CSV"按钮(每个表格独立按钮)
  - 导出走 Electron `dialog.showSaveDialog`,用户选位置
- i18n:`apps/desktop/src/renderer/i18n/zh-CN.json` + `en.json`,所有可见文案走 `t()` 函数
- `MainLayout`:三栏可拖拽(用 `react-resizable-panels`),Sidebar / MainWorkArea / RightPanel(右栏 v0.1 留空,M2 加浏览器预览)
- 拖拽宽度持久化(写 settings 表)

**架构决策(D-M1-7)**:
- M1 历史"恢复 session"只做查看(events 回放渲染),不重启 CC 子进程
- 真正"继续聊天"功能(`claude --resume <session-id>`)留 M2(F1.5)
- 这个简化让 M1 issue #12 工作量收敛

**验收**:
- 发送一个让 CC 输出包含 markdown 表格 + 代码块的回复,UI 渲染正确,代码块复制按钮工作,表格"导出 CSV"在 Excel/Numbers 能打开
- 输入框 Enter 发送、Shift-Enter 换行、粘贴图片自动 attach 后能发出去
- 完成 3 次会话后,历史栏显示 3 条记录,点击任一项,中栏完整渲染历史对话(events 回放)
- 设置语言切换中→英,所有静态文案立即更新
- 三栏宽度可拖拽,刷新后保持
- /settings/about 显示正确版本号 + 链接

**预估**:3-4 天(允许拆 12a / 12b)

---

### Issue #13:trade-email-writer skill + electron-builder 打包 + TD-002

**标签**:`milestone:M1`、`type:feat`、`P0`、`type:tech-debt`、`module:desktop`

**依赖**:前面所有(实际硬依赖 #6 / #7 / #9 / #11 / #12)

**描述**:

M1 收官 issue。三件事:
1. 实现 `skills/trade-email-writer` 完整内容
2. electron-builder 三平台打包配置 + CI 矩阵
3. TD-002:preload bundle 138KB 拆分

**要做**:

**part A — trade-email-writer skill**:
- `skills/trade-email-writer/skill.yml`:按 02 §四 Skill 3 + 03 §4.3 manifest 格式
  - inputs:scenario(select:报价/议价/催付/催货/售后/跟进)、context(textarea)、tone(select:formal/friendly/firm)、previousEmail(textarea, optional)
  - outputs:draftEmails、subjectLines、riskNotes
  - allowedTools:`mcp__opentrad__draft_save`、`Read`、`Write`、`WebSearch`(WebSearch 是 CC 内建)
  - disallowedTools:`Bash(*)`、`mcp__*__send*`(防御性)
  - stopBefore:无(本 skill draft_only 风险等级,不需业务级停止)
  - riskLevel:`draft_only`
- `skills/trade-email-writer/prompt.md`:外贸邮件写作 prompt 模板,覆盖 02 §四 6 类场景
  - 通用骨架:你是外贸邮件写作助手 + scenario 分支 + tone 分支
  - 6 个场景各给 2 个示例(few-shot)
  - 关键约束:不出现"作为 AI 助手"开场白、必须包含 FOB/CIF/MOQ 等行业术语(如适用)、生成 2-3 份不同角度的版本
- `skills/trade-email-writer/README.md`:用户可见说明
- `skills/trade-email-writer/icon.svg`:简洁 svg 图标(可暂用 emoji-style ✉️ 占位)
- `skills/trade-email-writer/examples/quotation.md` 和 `chasing-payment.md`:2 份示例输入输出
- 端到端验收测试:从 SkillPicker 选 trade-email-writer → 填"我想给越南买家催 $5000 货款,语气 firm,他们之前承诺 30 天内付,现在已 45 天" → CC 生成 2-3 份草稿
  - 每份英文地道(formal / friendly / firm 三语气可见区别)
  - 含 FOB/CIF/MOQ 等术语(如适用)
  - **绝无"作为 AI 助手"开场白**
  - 调用 draft_save 保存草稿到 `~/.opentrad/drafts/`

**part B — electron-builder 打包**:
- `apps/desktop/electron-builder.yml`:三平台配置
  - macOS:arm64 + x64(`.dmg` + `.zip`)
  - Windows:x64(`.exe` 安装包 + `.zip` portable)
  - Linux:x64(`.AppImage` + `.deb`)
  - extraResources:`apps/mcp-server` 编译产物 + Playwright Chromium(从 issue #10 配置接续)
  - 签名:暂留 TODO 注释(M3 配 macOS notarization + Windows code signing)
- `.github/workflows/release.yml`:tag 触发自动构建 artifact 上传
  - matrix:三平台
  - 触发条件:tag `v*` 或手动 `workflow_dispatch`
  - 产物上传到 GitHub Actions artifacts(**不公开 GitHub Release**,见 D-pre-2)
- 本地 `pnpm build:dist` 命令一键三平台打包(macOS 上 cross-compile Linux,Windows 走 CI)

**part C — TD-002 preload 拆分**:
- 把 `apps/desktop/src/main/preload.ts` 现有的 IpcChannels 类型按 domain 拆
  - `apps/desktop/src/common/ipc-channels/cc.ts`
  - `apps/desktop/src/common/ipc-channels/skill.ts`
  - `apps/desktop/src/common/ipc-channels/session.ts`
  - `apps/desktop/src/common/ipc-channels/risk-gate.ts`
  - `apps/desktop/src/common/ipc-channels/settings.ts`
  - `apps/desktop/src/common/ipc-channels/installer.ts`
  - `apps/desktop/src/common/ipc-channels/pty.ts`
- preload 只暴露 light surface(每个 domain 一个 namespace),大类型保留在 renderer 端引用

**验收**:
- `skills/trade-email-writer` 端到端跑完一次,产物:2-3 份邮件草稿(markdown 渲染,导出 .md 可读)
- 邮件英文地道(formal / friendly / firm 三语气可见区别),含 FOB/CIF/MOQ 等术语,无 AI 开场白
- 推一个 tag(如 `v0.1.0-rc.1`)触发 release.yml,3 平台 artifact 在 GitHub Actions 产物里能下载
- 下载 artifact 在三平台**真实物理机器**各装一台,完整跑通 onboarding(install → login → skill picker → trade-email-writer 生成草稿 → 导出 markdown)
- 应用体积:macOS .dmg 350-500MB、Windows .exe 350-500MB、Linux .AppImage 350-500MB
- preload bundle 体积从 138KB 降到 ≤80KB,typecheck 仍全绿
- M1 收官打 tag:`m1-complete`(发起人手工操作)

**预估**:2-3 天

---

## §三 决策点预测(可预见的 D-M1 编号决策)

参考 M0 的 D1/D6/D9 经验,M1 期间预计会触发若干"实战中即时拍板"的决策。提前列出可预见的几个,Claude Code 遇到时按 D9-1 判断标准:扩散到仓库之外的(影响未来贡献者、CI、用户)→ 停下来召唤总工 + 发起人;仅 CI / 单 package 内部 → 自主决策。

### 已在 issue 内部预先标注的决策

- **D-M1-1**:node-pty 在 Electron 主进程的 native module 重编译流程(issue #3,允许自主决策)
- **D-M1-2**:SkillLoader 加载失败 skill 的隔离策略(issue #6,已选 per-skill try/catch)
- **D-M1-3**:skill 输入表单的位置(issue #7,已选中栏内嵌)
- **D-M1-4**:mcp-server 是否常驻(issue #8,已选 per-task 拉起)
- **D-M1-5**:IPC bridge 连接失败时 mcp-server 行为(issue #8,已选 graceful degrade)
- **D-M1-6**:"编辑后再发" v1 语义(issue #11,已选 deny + 让 CC 重试)
- **D-M1-7**:历史 session "恢复"语义(issue #12,M1 只做查看回放,不重启 CC)

### 可能在施工时触发的额外决策

- **(可能) Onboarding 跳过路径**:用户选"我已装"但 detectInstallation 仍 false 时怎么办(issue #4)
- **(可能) Multi-session 并发的 IPC bridge socket 共用**:N 个 mcp-server 进程同时连同一 socket 的协议设计(M1 单 session,但 socket server 设计要为 M2 多 session 留口)
- **(可能) Risk Rule 唯一索引在 SQLite NULL 上的行为**:03 §五 schema 用 `COALESCE(skill_id, '')` 做 unique,M1 实施时验证 cross-platform SQLite 行为一致(issue #2 / #11)
- **(可能) Windows named pipe 路径冲突**:同名 OpenTrad 进程残留 pipe 时的清理(issue #8)
- **(可能) trade-email-writer prompt LLM 风格调优**:第一版 prompt 跑出来如果"AI 味"重,需要数轮迭代(issue #13)
- **(可能) electron-builder 三平台打包失败**:常见的是 native modules 在某平台 prebuilt binary 缺失,可能需要 `electron-rebuild` 或 `pnpm.supportedArchitectures` 调整(issue #13,M0 D9 同类问题)

---

## §四 时间预算 + M1 完成定义

### 时间预算

按 issue 列表逐个估算累加:

| Issue | 预估 |
|-------|------|
| #1 TD-D6 + 勘误 3 + 样本采集 | 1-1.5 天 |
| #2 SQLite 持久化层 | 2-3 天 |
| #3 PTY + xterm.js | 1.5-2 天 |
| #4 安装向导 + 检测强化 | 1.5-2 天 |
| #5 登录引导 | 1.5 天 |
| #6 Skill runtime 后端 | 1.5-2 天 |
| #7 Skill UI | 1.5 天 |
| #8 MCP server + IPC bridge | 3-4 天 |
| #9 MCP Config Writer + safe tools | 2 天 |
| #10 Browser tools + Chromium bundle | 2-3 天 |
| #11 Risk Gate 完整 | 3 天 |
| #12 Chat UI + History + Settings + Export | 3-4 天 |
| #13 trade-email-writer + 打包 + TD-002 | 2-3 天 |
| **合计** | **26-32 天** |

**保守预算 26-32 天**(对照 02-mvp-slicing 原 25-35 天,基本一致)。

**乐观预算**:按 M0 实际 3-4 天 / 预算 14-18 天的比率(约 1/4 速率),M1 乐观值 7-10 天。

**真实预期**:retrospective §二明确"不要按 M0 速度乐观推算 M1,M1 复杂度比 M0 高 2-3 倍"。**真实预期 12-18 天**——在乐观和保守之间,接近乐观侧。

实施过程中如某个 issue 实际显著超估算(>50%),Claude Code 应在 PR 描述里记录原因,回顾时进入 retrospective-m1。

### M1 完成定义

13 个 issue 全部 close + 13 个 PR 全部 merge + 满足以下硬指标:

1. **端到端用户路径跑通**:在三平台真实物理机器(macOS / Windows / Linux,最好不是开发机)各装一台,完成完整流程:首次启动 → onboarding(install CC → login Claude → skill picker)→ 选 trade-email-writer → 填表 → CC 生成 2-3 份邮件草稿 → 渲染 markdown → 导出 .md → 关闭重启后能从历史栏看到该 session
2. **三平台 artifact 打包成功**:推 tag 触发 release.yml,3 个 platform 的 artifact 都生成且可下载(GitHub Actions artifacts,非公开 release)
3. **应用体积**:三平台 .dmg / .exe / .AppImage 均在 350-500MB 范围
4. **CI 全绿**:M1 期间所有 PR 触发的 CI 三平台(lint / typecheck / test / build)全部通过
5. **关键性能指标(03 §十一)**:
   - 冷启动 ≤ 5 秒(M1 不强求 3 秒,M2 优化)
   - CC spawn 到首个事件 ≤ 3 秒
   - UI 响应 ≤ 200ms(M1 不强求 100ms)
6. **打 tag**:`m1-complete`(发起人手工操作)

### M1 不算完成的事

明确从 M1 完成定义中排除(避免范围蔓延):

- 公开 v0.1 release(release notes / 安装教程 / FAQ)——见 D-pre-2
- macOS notarization / Windows code signing——M3
- 其他 4 个 P0 skill 实现——M2
- 多 session 并发 / 真正的 session 续聊——M2
- 内置新手教程 / i18n 完整覆盖(M1 i18n 仅核心 UI 文案)——M3
- 性能调优到 03 §十一指标 100% 达标——M3

### 收官后

M1 收官时撰写 `retrospective-m1.md`,沉淀 D-M1 系列实际触发的决策、勘误、技术债、时间预算偏差。然后 M2 起草由独立 claude.ai 对话中的总工接手,本文档作为输入(同 M1 起草由 retrospective-m0.md 作为输入的模式)。

---

**正式标记 M1 起草完成,清单交付。开工权交回发起人,由发起人转发给 Claude Code 施工。**
