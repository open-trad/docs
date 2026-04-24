# OpenTrad 项目定位书 v1

**文档版本**：1.1  
**撰写日期**：2026-04-24  
**状态**：已审阅  
**作者**：总工（Claude Opus 4.7）

---

## 一句话定位

OpenTrad 是一个开源桌面应用，把 Claude Code 包装成外贸商家能上手的图形化 AI 工作台。

## 英文 Slogan

> **Open-source AI workbench for global traders.**  
> **Your Claude Code. Your workflow. Open forever.**

## 中文 Slogan

> **给外贸人的开源 AI 工作台**  
> **用你的 Claude Code，做你的生意，永远开源。**

---

## 一、为什么做这件事

跨境电商和外贸正在经历一场"AI agent 化"的工作方式变革。阿里国际站在 2026 年 3 月推出了 Accio Work，把多 agent 协作、skill 生态、浏览器自动化整合成一个桌面 App，面向 SME 外贸商家。这个产品方向是对的——外贸工作流（选品、询盘、发品、运营）天然适合 AI agent 来跑。

但 Accio Work 有三个硬伤：

第一，模型被网关锁死。所有 LLM 调用走阿里自家的 Phoenix 网关，用户看不到真实模型名（`1Nexus-xxx` / `1Drift-xxx` 这类混淆编号），没有 BYOK 入口，不能接自己的 API key，也不能用自己的 Claude 订阅。这意味着用户交的钱里既有产品费也有模型费，且无法优化。

第二，不开源。用户不知道数据怎么流转、skill 怎么执行、权限怎么判定。对外贸商家这个群体来说，商业敏感数据（供应商、客户、成本）交给一个黑盒应用存在合规隐患。

第三，垂直锁定阿里生态。Accio Work 的业务深度集中在 Alibaba.com / 1688，对 TikTok Shop、Shopify、Amazon 等其他平台支持薄。

**OpenTrad 的机会不是做一个更好的 Accio——而是做一个架构完全不同的替代品。**

## 二、我们做什么

OpenTrad 不自建 agent runtime，不做模型网关，不做任何"大脑"。

OpenTrad 是一个图形化宿主，把**用户自己的 Claude Code** 包装成外贸商家能用的工作台：

- 用户的 Claude 订阅（Pro / Max）直接复用，OpenTrad 不触碰 token 不转售不抽成
- 提供外贸场景的 skill 库（选品、RFQ、listing 本地化、HS code 等）
- 提供浏览器自动化、1688 搜索、供应商分析等工具（通过 MCP 供给给 Claude Code）
- 提供外贸商家真正需要的 Risk Gate：所有对外副作用动作（发 RFQ、发品、OAuth 授权、真实发信）默认停在最终确认前
- 完全开源（AGPL-3.0），商业模式是社区 + 未来可选的企业版/托管版

## 三、用户画像

### 核心用户：中小跨境电商/外贸商家

**典型画像**：  
独立站卖家 / 1688 档口店主 / TikTok Shop 小团队 / Amazon 铺货商家 / 阿里国际站中小商家。团队规模 1–20 人。年 GMV 几十万到几千万人民币。

**技术水平**：  
不是开发者，但不抗拒工具。用过 ChatGPT，可能订阅过 Claude，能看懂简单的配置向导，但不会写代码、不用命令行。

**核心诉求**：
- 找货：1688 选品、供应商评估、HS code 查询
- 谈判：RFQ 草稿、报价邮件、议价话术、催付跟进
- 上架：listing 本地化（中→英）、SEO 关键词、多平台适配
- 运营：竞品分析、店铺文案、客服回复

**付费意愿**：  
模型月费 20 美金可以接受（已付 Claude Pro 或 ChatGPT Plus），但不愿意再为一个工具软件付同量级订阅费。典型心态："你让我用我自己已经付的 AI，我就用；让我再单独付一份，我再想想"。

### 次核心用户：AI-savvy 的外贸 operator

已经在用 Claude Code / Cursor / Cline 做辅助工作的跨境业务负责人或独立运营者。他们不需要教育"什么是 AI agent"，但需要有人把 Claude Code 的能力**定向到他们的业务场景**——外贸 skill、浏览器控制、可视化任务面板。

### 暂不覆盖的用户

- **纯开发者**：他们直接用 Claude Code / Cursor 就够了，不需要 GUI
- **完全不会上网的商家**：装不了应用、配不了订阅、搞不定浏览器授权，这部分人不是 AI 产品能救的
- **企业级团队**：需要多人协作、审计、SSO、RBAC 的大团队暂缓，v1 先做单机工作台

## 四、核心场景（v1 首发）

根据调研，v1 发布时内置 5 个外贸 skill，覆盖外贸商家的核心工作流：

| Skill | 场景 | 一句话价值 |
|-------|------|-----------|
| `product-sourcing-1688` | 1688 选品找货 | 一段中文需求，返回候选商品清单 + 价格带 + MOQ + 供应商风险点 |
| `supplier-rfq-draft` | Alibaba 供应商询盘 | 贴个供应商链接，生成 RFQ 草稿 + 需澄清问题清单 |
| `trade-email-writer` | 外贸邮件写作 | 报价/议价/催付/跟进，一键生成符合行业调性的英文邮件 |
| `listing-localizer-seo` | 商品上架本地化 | 中文资料 → Amazon/Shopify/TikTok 英文 listing + SEO 关键词 |
| `hs-code-compliance-check` | 合规风险预检 | HS code 初查 + 禁限售 + 认证要求 + 报关描述草稿 |

这五个 skill 的共同特征：
- 离成交路径最近，商家能立刻感受到价值
- 外部副作用低（可以"停在发送前"），合规风险可控
- 不依赖平台账号深度集成，降低 v1 工程复杂度

具体 skill 实现方案见 Skill 设计文档（后续编写）。

## 五、差异化三角

OpenTrad 不是单纯对标某一个产品，而是在三个竞品象限里都吃一口：

```
            体验对标
             Accio
              ▲
              │
              │
   BYOK 开源  │  外贸垂直
   对标 Cline ◄─┼─► 对标龙虾
              │
              ▼
```

### 对标 Accio（体验层）

**Accio 有的我们要有**：多 agent 协作、skill 生态、浏览器自动化、任务看板、权限审批、向量记忆。

**Accio 没有的我们补上**：
- BYOK（真正让用户掌控模型）
- 开源（AGPL 协议、代码透明）
- 平台无关（不只 Alibaba/1688）
- Risk Gate（对副作用动作的业务级停止位）

### 对标 Cline（开源精神层）

**Cline 有的我们沿袭**：BYOK、本地执行、无服务器依赖、工具透明、开源社区共建。

**Cline 没有的我们补上**：
- 独立桌面应用（不绑 VS Code）
- 外贸垂直场景（不只 coding）
- 图形化 skill 市场（不需要看 markdown）
- Claude Code 订阅复用（不强求 API key）

### 对标龙虾 / 跨境通 / 店小秘等外贸 SaaS（场景层）

**他们有的我们学**：中文母语 UI、对 1688/Alibaba/TikTok Shop 的深度理解、外贸话术库、报关/HS code 知识。

**他们没有的我们补上**：
- 真正的 AI agent 能力（不是模板填空）
- 开源不锁定（数据不走他们服务器）
- 一次付费（Claude 订阅）打通所有模型，不按功能点付费

**结论**：OpenTrad 的独特定位是**三条线的交点**——这个交点上没有现成产品。

## 六、做什么不做什么（v1）

### 明确做的

- 把 Claude Code CLI 包装成桌面 GUI（结构化渲染对话、工具调用、任务进度）
- 自动化 Claude Code 的安装检测和登录引导（不代管凭证）
- 内置 MCP server，供给 Claude Code 外贸场景工具（浏览器、1688、listing 助手等）
- Skill 市场 v1：5 个 P0 skill 一键启用
- Risk Gate：业务级"最终确认前停止"
- 单机运行，所有数据本地，不走任何外部服务器（除 Claude 官方和用户自配的第三方 API）

### 明确不做的

- 不做自研 agent runtime（这是 Accio 的死胡同）
- 不做模型网关 / 付费代理（合规风险 + 无护城河）
- 不做用户账号系统 / 云端同步（v1 单机，v2 再考虑）
- 不做自动发送 / 自动发布 / 自动付款（副作用过大，永远停在确认前）
- 不做多人协作 / 团队版（v1 单用户单机）
- 不做非 Claude 模型（v1 只做 Claude Code 宿主，v2 考虑 Codex CLI 和国产模型直连）

### v1 之后再考虑的

- **v1.5**：Codex CLI 宿主（OpenAI 路径）、国产模型 BYOK（DeepSeek / 通义 / Moonshot）
- **v2**：企业版（团队协作、审计、SSO）、云同步、skill marketplace（社区贡献）
- **v3**：移动端（查看任务结果、远程审批）

## 七、商业模式

### v1：纯开源社区模式

- AGPL-3.0 协议
- 无付费功能、无订阅、无统计上报
- 运营成本极低（仅 GitHub / 域名 / 文档站）
- 推广靠口碑、内容营销、外贸社区渗透

### 未来可能的商业化路径（均不影响开源承诺）

1. **企业托管版**：给不想自己部署的团队提供云端版本（付费）
2. **官方 skill 扩展包**：行业深度包（比如特定品类的合规包、特定平台的集成包）按需付费
3. **API Reseller 分成**：引导用户开通 DeepSeek / Moonshot / Claude 时走官方联盟链接（零成本额外收益）
4. **咨询服务**：为大客户做定制 skill 开发和部署

**所有商业化都不会影响核心产品的开源承诺和单机可用性。** 这是对用户的承诺，写进 LICENSE 和 README。

## 八、开源许可和治理

### 许可证选择

| Repo | 许可证 | 理由 |
|------|--------|------|
| `opentrad/opentrad` | **AGPL-3.0** | 主应用。防止大厂 fork 闭源商用，强制衍生作品同样开源 |
| `opentrad/skills` | **MIT** | Skill 库。要鼓励社区贡献，越自由越好 |
| `opentrad/docs` | **CC-BY-4.0** | 文档。知识共享协议 |
| `opentrad/research` | 暂 private | 调研归档。v1 发布后再决定是否开放 |

### 治理模式

v1 阶段：**BDFL**（仁慈独裁者）模式。项目发起人（yrjm）作为唯一 maintainer，决定方向和合并。

社区贡献欢迎，但核心架构决定权集中，避免早期过度民主化导致方向分散。

v2 之后如果社区规模起来，可过渡到**技术委员会 + RFC 流程**。

---

## 附录 A：与 Accio Work 的详细对比

| 维度 | Accio Work | OpenTrad v1 |
|------|-----------|-------------|
| 模型来源 | 阿里 Phoenix 网关锁定 | 用户自己的 Claude 订阅 |
| 开源 | 否（闭源商业产品） | 是（AGPL-3.0） |
| 定价 | 订阅制 | 免费（只需用户自己的 Claude Pro） |
| 数据主权 | 走阿里服务器 | 完全本地 |
| Skill 生态 | 官方封闭 | 开源社区贡献 |
| 浏览器控制 | Relay 扩展 + CDP | CDP（v1）+ Relay（v2） |
| 权限模型 | policy + 沙箱 + 审批 | 同 + Risk Gate 外贸业务层 |
| 风险确认 | 工具级 | 工具级 + 业务级（发 RFQ/发品/付款分阶段） |
| 垂直深度 | Alibaba / 1688 强 | 多平台平衡（1688 / Alibaba / TikTok Shop / Shopify / Amazon） |
| 语言 | 中英双语 | 中英双语，中文优先 |
| 目标用户 | SME 外贸商家 | SME 外贸商家 + 独立 operator |

## 附录 B：名词对照表

| 英文 | 中文 | 说明 |
|------|------|------|
| Claude Code | Claude Code | 不翻译，保留英文 |
| Skill | 技能 / Skill | 中英混用 |
| MCP | MCP | 不翻译 |
| Risk Gate | 风险闸 / 确认闸 | 产品专有名词 |
| Process Manager | 进程管理器 | 架构模块名 |
| Stream Parser | 流解析器 | 架构模块名 |
| BYOK | BYOK / 自带 key | Bring Your Own Key |