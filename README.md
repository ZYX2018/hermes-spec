# Hermes-Spec

**Hermes-Spec** 是一套面向 **OpenSpec 工作区** 的 AI 工程化内核：通过 `npm install -g hermes-spec`，把 Agent 工作流编排、独立评审智能体、规范模板、门禁技能与本地记忆体系**一键下发**到任意业务代码仓——且**不覆盖**你已有的 PRD、change、spec 与自定义 rule。

> 本仓库（GitHub · 公开）托管 **AI 可执行的安装指引** 与项目介绍，供社区试用与反馈。完整 npm 包与内核源码在内网/私有发行渠道维护。

---

## Hermes-Spec 是什么？

在 Cursor、CodeBuddy 等 IDE 里，大模型可以写代码，但复杂需求往往缺少**可审计的工程闭环**：PRD 怎么定稿、设计怎么评审、实现怎么与需求对齐、验收证据怎么留存、团队经验怎么沉淀。

Hermes-Spec 在 [OpenSpec](https://github.com/Fission-AI/OpenSpec) 之上补一层 **「规格驱动 + 多 Agent 编排 + 门禁证据」**：

| 层次 | 做什么 |
|------|--------|
| **OpenSpec** | change / proposal / specs / tasks 等规格工件与 CLI |
| **Hermes-Spec** | 产品 PRD 流、反交底、Hermes propose/apply、Clarify/Verify 门禁、评审智能体、记忆与技能蒸馏 |
| **业务仓** | 你的 Java / Go / Vue 等项目代码；Hermes 只下发 `.cursor/`、`.codebuddy/`、`openspec/` 中的**增量资产** |

一句话：**让 AI 按软件工程纪律干活，而不是「聊完就写、写完就算」。**

---

## 核心亮点

### 1. Agent 工作流编排

Hermes 把复杂需求拆成**可重复、可门禁**的阶段链路，由 Skills 与角色卡驱动 IDE 内 AI 执行：

```text
产品 PRD（draft → update → lock）
    ↓
研发反交底（PRD × 代码支持度矩阵，复杂单 MUST）
    ↓
/opsx:hermes-propose（渐进接地 → 子代理规划 → Clarify 门）
    ↓
/opsx:hermes-apply（Dev → 独立 CR → 独立 TE → 验收评审）
    ↓
人工 Accept → Verify 门禁 → Archive → 记忆睡眠
```

- **双轨选用**：小单走 OpenSpec 默认 `spec-driven`；跨模块、多待确认、有正式 PRD 的复杂单走 **`subagent-spec-driven`** + FIC 子代理规划。
- **Clarify 硬门**：规划未关「待确认」禁止 Apply；环境 BLOCKED 与文本待确认分离判定。
- **Verify 证据门**：Archive 前须有测试/评审/验收产物；行为变化须回写 spec，旧证据失效则重验。
- **三权分离（代码阶段）**：实现 Dev、代码评审 CR、测试 TE 使用**独立上下文**，避免「自己写、自己评、自己改」的盲区继承。

典型入口命令（安装后在 IDE 对话中使用）：

| 命令 | 用途 |
|------|------|
| `/opsx:hermes-prd-draft` | 基于代码现状起草 PRD |
| `/opsx:hermes-req-handback` | 研发反交底：需求×代码对照矩阵 |
| `/opsx:hermes-propose` | 复杂变更：规划 + 设计 + specs + tasks |
| `/opsx:hermes-apply` | 实现 + 独立评审/测试闭环 |
| `/opsx:archive` | 人工验收通过后归档 |

---

### 2. 独立评审智能体（前段 + 末端）

除代码阶段的 CR/TE 外，Hermes 在 **PRD、反交底、设计、验收** 四个节点挂载独立评审角色，评判标准统一维护于 `review-standards.md`（ALG-01～05），输出可审计的 `review-loop.md` 与轮次报告：

| 评审节点 | 对照基线 | 价值 |
|----------|----------|------|
| **requirement-reviewer** | 需求完整性 / ISO 29148 | 编码前消化 PRD 缺口 |
| **handback-reviewer** | PRD×代码支持度 | 避免「文档写了、代码没有」 |
| **design-reviewer** | design / spec / proposal 联动 | 设计评审可组会、可留痕 |
| **acceptance-verifier** | **锁定 PRD** 需求覆盖 | 验收从「口头 OK」变为覆盖矩阵 + Verdict |

验收 Verdict 四级（`COMPLETE` / `PASS_WITH_GAPS` / `BLOCKED` / `FAIL`）：⚠️ 向用户请求与 ❌ 交回 Dev 语义隔离，减少 silent pass。

---

### 3. 本地记忆与渐进接地

Hermes 不依赖模型「记住上次聊过什么」，而是把**可检索的工程记忆**写进仓库：

- **双 index**：`openspec/specs/index.md`（现行规范索引）+ `openspec/changes/archive/index.md`（已归档变更与关键决策摘要）。
- **渐进接地（L0→L1→L2）**：先查 index → 读命中正文 → 再 codegraph / RAG 深查，控制 token 与幻觉。
- **Memory Sleep（记忆睡眠）**：每次 Archive 后自动整理**周卡 + 滚动主文档**，向已配置的持久记忆提供者发出存储/更新意图（Skill 或 MCP，不绑定固定工具名）。
- **RAG 可选**：`.hermes-spec/rag.yaml` 配置 `providerType` + `providerName`（如 rag-cli 技能），与 OpenSpec config 隔离；`doctor` 报告就绪状态。

---

### 4. 技能沉淀（经验 → 可复用 SOP）

踩过的坑不应只留在聊天记录里：

1. **Archive 后**：`hermes-memory-sleep` 将可复用 SOP 草稿写入 `openspec/memory/skill-drafts/`。
2. **人确认启用**：`hermes-distill-skill` 把草稿提升为正式 Skill，写入各合格 IDE 的 `skills/<name>/SKILL.md`。
3. **团队传播**：下次同类任务 Agent 自动加载 Skill，减少重复探索与口径漂移。

这是「个人经验 → 团队资产」的闭环，且**必须经人批准**才会写入 IDE 技能目录。

---

### 5. 安装友好 · 棕地安全

- **一条龙**：`hermes-spec init` = 检测 OpenSpec → init → 全量安装 Hermes 资产。
- **不覆盖用户资产**：已有 `openspec/changes/`、`openspec/specs/`、自定义 rule、部门编码规范等**跳过**，只补缺失。
- **双 IDE 扇出**：一次安装同步 Cursor + CodeBuddy（同 MANIFEST 哈希）。
- **doctor 七维就绪**：standards 插槽、docs 断链、config skeleton、codegraph、md2word、RAG、PRD 目录等。

---

## 快速开始

### 给谁用？

| 角色 | 建议 |
|------|------|
| **人类开发者** | 把 [HERMES-SPEC-AI-INSTALL.md](./HERMES-SPEC-AI-INSTALL.md) 交给 IDE 里的 AI，让它在你的**业务仓**完成安装 |
| **AI Agent** | 直接阅读安装文档并严格按 Scenario 执行（环境探测 → 分场景路由 → 记录 exit code） |

### 环境要求

- **Node** ≥ 20.19.0  
- **OpenSpec CLI** effectiveMin ≥ 1.6.0  
- **IDE**：Cursor 或 CodeBuddy（推荐）

### 安装示例

```powershell
# 1. 全局安装 CLI
npm install -g hermes-spec

# 2. 在目标业务仓接入（路径替换为你的业务仓绝对路径）
hermes-spec init --yes --target C:\path\to\your-business-repo
hermes-spec doctor --target C:\path\to\your-business-repo
```

👉 完整分场景指引（全新接入 / 棕地升级 / 更新 / 卸载 / Monorepo 陷阱）：  
**[HERMES-SPEC-AI-INSTALL.md](./HERMES-SPEC-AI-INSTALL.md)**（当前对齐 `hermes-spec@0.6.1+`）

---

## 本仓库有什么

| 文件 | 说明 |
|------|------|
| [README.md](./README.md) | 项目介绍（本页） |
| [HERMES-SPEC-AI-INSTALL.md](./HERMES-SPEC-AI-INSTALL.md) | AI 安装 / 更新 / 卸载主文档 |

安装文档内含：

- AI 执行契约（先探测后分支、非 0 即停、命令必须真实执行）
- Phase 0 环境探测与 Scenario 路由表
- doctor 报告字段解读（RAG、md2word 等）
- Windows / PowerShell 与 Monorepo 注意事项

---

## 反馈与贡献

欢迎通过 **[Issues](https://github.com/ZYX2018/hermes-spec/issues)** 反馈：

- 某 IDE / 模型下安装步骤走不通  
- 文档 Scenario 缺失或表述歧义  
- 对 Agent 工作流、评审、记忆设计的建议  

请尽量附上：**操作系统、Node 版本、IDE、完整命令、exit code、输出末 20 行**。

---

## 仓库关系

| 仓库 | 用途 |
|------|------|
| **本仓库（GitHub · 公开）** | 项目介绍 + AI 安装指引 · 社区反馈 |
| **内核仓（私有/内网）** | npm 包、skills/agents 权威源、OpenSpec 规范、CI 与发布门禁 |

文档会不定期从内核仓同步；版本与行为以安装文档文内说明为准。

---

## 许可与声明

Hermes-Spec 内核包与配套资产由作者团队维护。本公开仓库仅用于**文档传播与安装体验反馈**；npm 包的获取与商业使用条款以内网/发行渠道说明为准。
