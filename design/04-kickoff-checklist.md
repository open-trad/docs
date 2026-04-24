# OpenTrad 开工前准备清单

**文档版本**：1.2（最终版）  
**撰写日期**：2026-04-24  
**状态**：待执行  
**作者**：总工（Claude Opus 4.7）  
**依赖**：`01-positioning.md` v1.1、`02-mvp-slicing.md` v1.0、`03-architecture.md` v1.0

---

## 本文档的用途

这份是三件套之后、Claude Code 开工之前的**最后一份筹备文档**。完成这份清单后，项目就进入施工阶段。

清单内容：
1. 门面文件的完整内容（Claude Code 施工时照抄）
2. 第一批 GitHub Issues（覆盖 M0 骨架可跑里程碑）
3. 给 Claude Code 的第一个施工 prompt

---

## 角色分工

- **项目发起人（yrjm）**：产品决策、测试验收、GitHub 上的关键手工操作（建 repo、合 PR、重大分支操作等）
- **Claude Code**：本项目的**唯一施工单位**。可以调用自己的所有工具（包括 codex-skill 作为能力之一），按设计文档施工
- **总工（Claude Opus 4.7）**：在独立的 claude.ai 对话里提供架构和设计支持。日常施工中不直接出现，发起人遇到架构级问题时新开对话召唤

**术语澄清**：文档里出现的 **Codex**（大写开头、无限定）指 **OpenAI Codex CLI**，一个独立的命令行 coding agent 产品，是未来 v1.5 的宿主候选。**codex-skill**（小写、带 `-skill` 后缀）指 Claude Code 自身的一个 skill，是 Claude Code 的能力之一。两者完全不同。

---

## 一、当前状态（已完成）

以下步骤发起人已经完成，Claude Code 开工时不需要重做：

- ✅ GitHub org `open-trad` 已创建
- ✅ 主仓库 `open-trad/opentrad` 已创建，private 可见性
- ✅ 主仓库已通过 GitHub 网页添加 `LICENSE` 文件（AGPL-3.0），当前 repo 有 1 个 commit
- ✅ 设计文档 repo `open-trad/docs` 已创建，三件套已推送
- ✅ 本地工作目录 `~/Desktop/open-trad/opentrad/` 已 clone
- ✅ 在工作目录里已启动 `claude`

**Claude Code 开工时的第一件事**：验证上述状态，特别是 LICENSE 文件是否是完整的 AGPL-3.0 全文（见 §2.1）。

---

## 二、门面文件

除了已经存在的 `LICENSE`，以下文件全部由 Claude Code 在施工前创建。内容完全照抄，不修改不加内容。

### 2.1 `LICENSE`（已存在，需校验）

发起人已通过 GitHub 网页创建 AGPL-3.0 LICENSE。Claude Code 开工时校验：

```bash
# 检查第一行是否包含 "GNU AFFERO GENERAL PUBLIC LICENSE"
head -1 LICENSE
# 预期输出：                    GNU AFFERO GENERAL PUBLIC LICENSE

# 检查行数（完整 AGPL-3.0 应在 660 行左右）
wc -l LICENSE
# 预期输出：约 660 行

# 检查是否包含 "Version 3, 19 November 2007"
grep "Version 3" LICENSE
```

如果任一项校验失败（比如只有 SPDX 标识符的短版本），从 GNU 官方拉完整版覆盖：

```bash
curl -o LICENSE https://www.gnu.org/licenses/agpl-3.0.txt
```

如果校验通过，继续下一步。

### 2.2 `README.md`

```markdown
# OpenTrad

**Open-source AI workbench for global traders.**  
**Your Claude Code. Your workflow. Open forever.**

OpenTrad 是一个开源桌面应用，把 Claude Code 包装成外贸商家能上手的图形化 AI 工作台。

## 为什么做 OpenTrad

跨境电商和外贸正在经历"AI agent 化"的工作方式变革。现有产品要么封闭（Accio Work 锁阿里模型生态）、要么太技术（Claude Code / Cursor 是给开发者用的）、要么太浅（传统外贸 SaaS 只是模板填空）。

OpenTrad 的定位是**三条线的交点**：
- 体验像 Accio，开箱即用、skill 生态、浏览器自动化
- 哲学像 Cline，BYOK、本地执行、完全开源、数据自主
- 场景像龙虾，中文母语、懂 1688 / Alibaba / TikTok Shop

## 核心特性

- **不碰用户凭证**：用户自己的 Claude 订阅，OpenTrad 只调用不代管
- **外贸场景 skill 库**：1688 选品、RFQ 草稿、邮件写作、listing 本地化、HS code 合规
- **浏览器自动化**：通过 MCP 给 Claude Code 提供浏览器控制工具
- **Risk Gate**：对发送、发品、付款等副作用动作的业务级确认
- **完全本地**：数据不离开你的电脑，不做 telemetry，不做云同步

## 状态

🚧 v1 开发中，暂不对外发布。设计文档见 [open-trad/docs](https://github.com/open-trad/docs)。

## 许可证

[AGPL-3.0](LICENSE) © 2026 OpenTrad contributors

AGPL 的含义：你可以自由使用、修改、分发 OpenTrad，但任何修改版本必须同样开源。这是为了防止大厂 fork 闭源商用。

## 贡献

见 [CONTRIBUTING.md](CONTRIBUTING.md)。目前处于 BDFL（仁慈独裁者）阶段，核心方向由项目发起人决定。欢迎 issue 和 discussion，PR 请先提 issue 讨论。
```

### 2.3 `CONTRIBUTING.md`

```markdown
# 贡献指南

感谢你对 OpenTrad 感兴趣。这份文档说明如何给项目贡献。

## v1 阶段的治理模式

OpenTrad v1 处于 BDFL（Benevolent Dictator For Life）阶段，项目发起人（yrjm）作为唯一 maintainer，核心架构决策由其单人决定。这不是因为社区不重要，而是早期项目需要集中决策保证方向一致。

v2 之后，如果社区规模起来，会过渡到技术委员会 + RFC 流程。

## 可以怎样贡献

### 报告 bug

在 [Issues](https://github.com/open-trad/opentrad/issues) 提交 bug 报告，附上：
- 操作系统和版本
- OpenTrad 版本（在"关于"里查看）
- Claude Code 版本（`claude --version`）
- 复现步骤
- 期望行为 vs 实际行为
- 截图或日志（记得脱敏）

### 提建议

在 [Discussions](https://github.com/open-trad/opentrad/discussions) 提功能建议，**不要直接提 PR 加新功能**——先讨论方向。

### 贡献代码

1. 先在 Issues 或 Discussions 讨论，确认方向被接受
2. Fork 仓库，创建 feature branch（命名：`feat/xxx` 或 `fix/xxx`）
3. 按 [开发指南](#开发指南)本地跑通
4. 提交 PR 到 `main` 分支，PR 描述说明：改了什么、为什么、怎么测试的
5. CI 通过 + maintainer review → 合并

### 贡献 skill

Skill 是独立的 markdown + yaml 包，贡献到 [open-trad/skills](https://github.com/open-trad/skills) repo，不是这个主仓库。

### 贡献文档

文档在 [open-trad/docs](https://github.com/open-trad/docs)。

## 开发指南

（待 M0 骨架完成后补充）

## 提交规范

Conventional Commits：

- `feat: 新功能`
- `fix: 修 bug`
- `docs: 文档修改`
- `chore: 杂项（依赖升级、配置调整）`
- `refactor: 重构，不改外部行为`
- `test: 加测试`
- `perf: 性能优化`

## 行为准则

对他人友善，讨论技术不讨论立场。违反者 maintainer 有权锁讨论或拉黑。

## License

贡献到本仓库的代码默认以 [AGPL-3.0](LICENSE) 授权。
```

### 2.4 `.gitignore`

```gitignore
# Dependencies
node_modules/
.pnpm-store/

# Build outputs
dist/
out/
build/
*.tsbuildinfo

# Electron
release/
app-builds/

# Logs
logs/
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Editor
.vscode/
.idea/
*.swp
*.swo
.DS_Store
Thumbs.db

# Env
.env
.env.local
.env.*.local

# Test coverage
coverage/
.nyc_output/

# Temporary
tmp/
temp/
.cache/

# OS
Icon?
ehthumbs.db

# Claude Code artifacts（项目级，不污染主仓库）
.claude/
.mcp.json

# OpenTrad runtime data（如果开发时生成到项目目录）
.opentrad/
```

### 2.5 `SECURITY.md`

```markdown
# 安全政策

## 支持版本

v1 开发阶段，暂无 stable release。所有 security fix 只会在 `main` 分支。

## 报告漏洞

**不要**在公开 Issue 里报告安全漏洞。

请在 GitHub 的 Security 页面使用 [Private Vulnerability Reporting](https://github.com/open-trad/opentrad/security/advisories/new) 提交。

报告应包含：
- 漏洞描述
- 复现步骤
- 影响范围
- 建议修复方案（可选）

我们会在 48 小时内确认收到，并保持沟通直到修复。

## 已知的安全边界

OpenTrad 的安全模型假设：
- 用户的操作系统未被攻破
- 用户的 Claude Code 凭证由 Claude Code 管理（Keychain 等），OpenTrad 不读取
- 用户对自己安装的第三方 skill 的代码负责

OpenTrad 不承诺防御：
- 物理接触攻击
- 操作系统级恶意软件
- 用户主动分享 `.opentrad/` 目录给不可信第三方导致的数据泄漏
```

### 2.6 `.editorconfig`

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

### 2.7 `.nvmrc`

```
20.18.0
```

（Node 20 LTS，Electron 34 和 Claude Code 都兼容）

### 2.8 `package.json`（根 monorepo）

```json
{
  "name": "opentrad-monorepo",
  "version": "0.0.0",
  "private": true,
  "description": "Open-source AI workbench for global traders",
  "repository": {
    "type": "git",
    "url": "https://github.com/open-trad/opentrad.git"
  },
  "license": "AGPL-3.0-only",
  "author": "OpenTrad contributors",
  "engines": {
    "node": ">=20.18.0",
    "pnpm": ">=10.0.0"
  },
  "packageManager": "pnpm@10.0.0",
  "scripts": {
    "dev": "pnpm --filter @opentrad/desktop dev",
    "build": "pnpm -r build",
    "lint": "biome check .",
    "format": "biome format --write .",
    "typecheck": "pnpm -r typecheck",
    "test": "pnpm -r test"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.0.0",
    "typescript": "^5.5.0"
  }
}
```

### 2.9 `pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### 2.10 `biome.json`

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

### 2.11 `tsconfig.base.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": false
  }
}
```

### 2.12 Issue / PR 模板

创建 `.github/` 目录，包含：

**`.github/ISSUE_TEMPLATE/bug_report.md`**：
```markdown
---
name: Bug Report
about: 报告一个 bug
labels: ["bug"]
---

## 环境

- 操作系统：
- OpenTrad 版本：
- Claude Code 版本（`claude --version`）：

## 描述

简要描述 bug。

## 复现步骤

1. 
2. 
3. 

## 期望行为

## 实际行为

## 附加信息

截图、日志（记得脱敏）。
```

**`.github/ISSUE_TEMPLATE/feature_request.md`**：
```markdown
---
name: Feature Request
about: 提一个功能建议
labels: ["enhancement"]
---

## 问题

这个功能解决什么问题。

## 建议

你认为怎么做比较好。

## 替代方案

你考虑过的其他方案。

## 额外信息
```

**`.github/pull_request_template.md`**：
```markdown
## 改了什么

## 为什么

## 怎么测试

## Checklist

- [ ] 本地 `pnpm lint` 通过
- [ ] 本地 `pnpm typecheck` 通过
- [ ] 本地 `pnpm test` 通过
- [ ] 有对应的 issue（#xxx）
- [ ] 文档已更新（如适用）
```

### 2.13 首次 commit 推送

全部创建完成后：

```bash
git add .
git commit -m "chore: add readme, contributing, tooling and templates"
git push origin main
```

---

## 三、M0 的第一批 Issues

M0 的目标是"骨架可跑"：应用能打开，能 spawn Claude Code，能渲染 hello world 级别的对话。

把 M0 的 6 个 P0 功能拆成 **8 个 issue**。Claude Code 用 `gh issue create` 批量创建，按依赖顺序排列。

### Issue #1：初始化 monorepo 骨架

**标签**：`milestone:M0`、`type:infra`、`P0`

**描述**：
按 `03-architecture.md` 第二章"monorepo 结构"创建完整目录骨架。

**要做**：
- 在 repo 根创建 `apps/`、`packages/`、`skills/`、`docs/`、`scripts/`、`.github/workflows/` 目录
- 为每个计划中的 package 创建空的 `package.json` 和 `tsconfig.json`（内容先占位，后续 issue 填充）
- `pnpm install` 能跑通，不报错

**验收**：
- `pnpm install` 成功
- `pnpm -r typecheck` 成功（空 package 也能通过）
- `pnpm lint` 成功

**预估**：1 天

### Issue #2：实现 packages/shared 的类型定义

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:shared`

**依赖**：#1

**描述**：
实现 `packages/shared`，存放跨包共享的类型和常量。按 `03-architecture.md` 第四章各个模块接口的类型定义，抽出共享部分：

- `CCEvent`（所有 stream-json 事件的 discriminated union）
- `CCTaskOptions` / `CCTaskHandle`
- `SkillManifest` / `SkillInput`
- `RiskGateRequest` / `RiskGateDecision`
- 各 IPC channel 的 payload 类型

**验收**：
- `packages/shared/src/types/*.ts` 完整
- 从其他包可以 `import { CCEvent } from '@opentrad/shared'`
- 有基本的单元测试（用 zod 做 schema 校验的测试）

**预估**：1-2 天

### Issue #3：实现 packages/stream-parser 的 NDJSON 解析

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:stream-parser`

**依赖**：#2

**描述**：
按 `03-architecture.md` 4.2 节实现 `StreamParser`：

- 接收 string chunk，yield `CCEvent`
- 支持 chunk 跨行拼接（buffer）
- 未知 type 转成 `{ type: 'unknown', raw: ... }` 不丢弃
- 实现 schema version guard（`COMPAT_MATRIX`）

**验收**：
- 单元测试覆盖：正常事件解析、chunk 跨行、不完整 JSON、未知类型
- 用真实的 CC `stream-json` 输出样本（从 `zdywrnm/acciowork-research` repo 的 `07-raw-evidence/cc-local-cli-probe.redacted.md` 里扒一份）做端到端解析测试
- `pnpm --filter @opentrad/stream-parser test` 全绿

**预估**：2-3 天

### Issue #4：实现 packages/cc-adapter 的进程管理

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:cc-adapter`

**依赖**：#2、#3

**描述**：
按 `03-architecture.md` 4.1 节实现 `CCManager`：

- `detectInstallation()`：跑 `claude --version`，parse 版本号
- `getAuthStatus()`：跑 `claude auth status --text`，parse 登录状态
- `startTask(opts)`：spawn `claude -p --output-format stream-json ...`，返回 `CCTaskHandle`
- handle 通过 async iterable 暴露事件流
- 应用退出时 `cleanup()` 清理所有子进程

**注意**：
- **不要读取任何 Claude 凭证文件**
- 本地测试时先跑通简单 prompt 如 `"Say OK"`，别用复杂 skill

**验收**：
- 单元测试（mock child_process）
- 手工端到端测试：写一个最小脚本，调用 `CCManager.startTask({ prompt: "Say OK" })`，打印所有事件，能看到 system/init → assistant text → result 完整流
- 子进程在应用退出时被正确清理（`ps aux | grep claude` 验证）

**预估**：3-4 天

### Issue #5：搭 Electron 主进程骨架

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:desktop`

**依赖**：#1

**描述**：
按 `03-architecture.md` 目录结构，在 `apps/desktop/` 搭 Electron 骨架：

- `electron.vite.config.ts` 配置
- 主进程入口 `src/main/index.ts`（创建 BrowserWindow，加载 renderer）
- 预加载脚本 `src/main/preload.ts`（contextBridge）
- 渲染进程入口 `src/renderer/index.tsx`（最小 React 页面，显示"Hello OpenTrad"）
- `pnpm dev` 能启动应用、看到窗口

**验收**：
- `pnpm --filter @opentrad/desktop dev` 成功启动
- 窗口标题正确显示 "OpenTrad"
- React devtools 工作正常
- 关闭窗口后进程干净退出

**预估**：2 天

### Issue #6：实现基本的 IPC 通信

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:desktop`

**依赖**：#5

**描述**：
按 `03-architecture.md` 第三章 IPC 协议设计，实现主进程和渲染进程的 IPC 基础设施：

- `src/common/ipc-channels.ts`：所有 channel 常量和 payload 类型（抄 `@opentrad/shared`）
- `src/main/ipc/`：每个 domain（cc / skill / session / settings）一个 handler 文件
- `src/main/preload.ts`：用 `contextBridge` 暴露 typed API 给 renderer
- 实现首个 channel：`cc:status`，renderer 能调用 `window.api.cc.status()` 拿到当前 CC 状态（调用 #4 的 `CCManager`）

**验收**：
- 在 renderer 的 React 组件里调用 `window.api.cc.status()` 能拿到 `{ installed: true/false, version?, loggedIn?, email? }`
- TypeScript 类型一路到底，renderer 端能看到完整类型提示
- IPC 失败（比如 CC 未安装）有友好错误提示

**预估**：2 天

### Issue #7：Hello World 端到端 demo

**标签**：`milestone:M0`、`type:feat`、`P0`、`module:desktop`

**依赖**：#3、#4、#5、#6

**描述**：
把前面所有模块串起来：用户点一个按钮，应用 spawn Claude Code 说"Say hi in Chinese"，stream-json 事件流式渲染到 UI。

**要做**：
- UI 上一个 `<button>` "Say Hi"，点击后发 IPC `cc:start-task`
- 主进程收到后调用 `CCManager.startTask({ prompt: "Say hi in Chinese" })`
- 事件流通过 IPC `cc:event` 推送回 renderer
- Renderer 用最简单的方式渲染：`assistant` text 显示为气泡，`tool_use` 显示为占位卡片，`result` 显示为"完成"标记
- 不做 markdown 渲染（那是 M1 的事），就纯文本
- 不做 skill，不做 MCP，不做 Risk Gate（都是后续里程碑）

**验收**：
- 点击按钮 → 2 秒内看到第一个事件 → 完整对话显示出来 → 看到 "完成"
- 再点一次能跑第二次（session 管理正确）
- CC 未安装或未登录时，按钮不可点击，有友好提示

**预估**：2-3 天

**这个 issue 完成 = M0 完成 = 骨架可跑。**

### Issue #8：CI/CD 基础配置

**标签**：`milestone:M0`、`type:infra`、`P0`

**依赖**：#1

**描述**：
在 `.github/workflows/` 下创建：

- `ci.yml`：PR 时跑 `pnpm lint` + `pnpm typecheck` + `pnpm test`
- macOS / Windows / Linux 三平台 matrix
- Node 20 版本锁

**不做**：
- electron-builder 打包（M1 再加）
- 自动发布（M1 再加）

**验收**：
- 本 issue 的 PR 本身触发 CI 并通过
- CI 跑 3 平台都成功
- 有状态徽章放到 README 底部

**预估**：1 天

### M0 汇总

8 个 issue，总工作量 **12-18 天**。对应定位书里的 "M0 骨架可跑 7-10 天" 预算——略超，但多加了 CI 和 IPC 基础设施这两项，值得。

---

## 四、给 Claude Code 的 kickoff prompt

在已启动的 `claude` 会话里，把下面这段作为第一条消息发给它：

---

```markdown
你好，我是 OpenTrad 项目的发起人。

## 背景

OpenTrad 是一个开源桌面应用，把 Claude Code 包装成外贸商家能用的图形化 AI 工作台。设计三件套 + 开工清单已经写完，存在 GitHub `open-trad/docs` repo（private，我的账号有权限）。

## 你的任务

你是这个项目的**唯一施工单位**。你可以调用你自己的所有工具（包括 codex-skill 等 skill 作为能力之一）。架构、设计决策、要做什么功能都已经定了，你按设计文档施工，不自由发挥。

## 当前工作目录状态

`~/Desktop/open-trad/opentrad/` 是已经 clone 好的主仓库，当前状态：
- `main` 分支有 1 个 commit：通过 GitHub 网页添加了 LICENSE 文件
- LICENSE 内容是 AGPL-3.0（需要你验证是否为完整全文，约 660 行）
- 除 LICENSE 外没有任何其他文件

## 开工流程

**第一步：读完四份文档**。从 `open-trad/docs` repo 拉：
- `design/01-positioning.md` — 项目定位
- `design/02-mvp-slicing.md` — MVP 切片
- `design/03-architecture.md` — 架构设计
- `design/04-kickoff-checklist.md` — 本项目的开工清单

用 `gh repo clone open-trad/docs /tmp/opentrad-docs` 或类似方式拉下来读。

读完后跟我确认三件事：
1. OpenTrad 是什么（一句话复述）
2. 你看懂了 monorepo 结构和 5 大模块
3. 你明白 M0 的 8 个 issue 依赖关系

**第二步：开工前问答**。读完文档后，不要立即写代码，先告诉我：
1. 任何你觉得设计文档里写的有问题、矛盾、或该讨论的技术决策
2. 任何 issue 拆分你觉得不合理的地方
3. 你需要我做什么准备（环境、密钥、授权）

**第三步：验证并初始化门面文件**。
1. 先校验 `LICENSE` 是否是完整的 AGPL-3.0（见 `04-kickoff-checklist.md` §2.1 的校验步骤）
2. 按 §2.2-§2.12 创建全部其他门面文件
3. `git add . && git commit -m "chore: add readme, contributing, tooling and templates" && git push`

**第四步：创建 GitHub Issues**。按 `04-kickoff-checklist.md` 第三章，在 `open-trad/opentrad` 仓库用 `gh issue create` 批量创建 8 个 issue。title、body、labels 照抄。

**第五步：从 Issue #1 开始施工**。按依赖顺序推进。

## 规则

- 完全按 `03-architecture.md` 的结构和命名来，不自由发挥
- 每完成一个 issue：commit（Conventional Commits）、push、在 issue 下评论"done"、关闭 issue
- 遇到设计文档没覆盖的问题，停下来问我，不要自己瞎决定
- 想挑战某个架构决策（比如"用 Jotai 比 Zustand 好"），说服我后再改，不要默默改
- 所有代码注释用中文，所有 log 用英文（方便国际化）
- 不读取 `~/.claude/` 目录，不读取 Keychain，不访问任何我没授权的本地路径
- 涉及架构级疑问或跨文档的权衡问题，告诉我"需要召唤总工"，我会开新对话找总工（Claude Opus 4.7）确认

准备好了吗？先读文档，然后回我第二步的问答。
```

---

## 五、发起人接下来要做的

**就一件事**：把上面第四章的 kickoff prompt 复制、粘贴到已启动的 `claude` 会话里，发送。

之后就是监工：
- Claude Code 问你问题时及时回答
- Claude Code 提出架构挑战时判断是否采纳（如果不确定，新开 claude.ai 对话找总工）
- 每个 issue 完成后验收

---

## 六、后续里程碑的准备

M0 完成后：
- M1 再开 12 个 issue（v0.1 preview）
- M2 再开 12 个 issue（v0.5 Beta）
- M3 再开 11 个 issue（v1.0 GA）

每个里程碑的 issue 清单等 **M0 跑通**后由总工单独起草。不要一次性把 43 个 issue 全开了，因为：
1. 早期的理解可能会影响后面的设计
2. issue 积压太多会让贡献者迷失重点
3. 每个里程碑发布前做一次 retrospective，调整后续计划
