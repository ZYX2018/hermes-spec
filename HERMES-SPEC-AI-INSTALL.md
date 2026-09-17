# Hermes-Spec AI 安装 / 更新 / 卸载指引

> **读者**：Cursor、CodeBuddy 等 IDE 内的 AI Agent（含 DeepSeek、Kimi、GPT、Grok、Qwen 等）。  
> **目标**：在用户指定**目标仓**（业务代码仓库）完成 Hermes-Spec 的安装、更新或卸载。  
> **版本**：对齐 npm 包 `hermes-spec@0.6.1+`（Node ≥ 20.19.0，OpenSpec CLI effectiveMin ≥ 1.6.0）。

---

## 0. AI 执行契约（所有模型 MUST 遵守）

1. **先探测、后分支**：未完成 §1 环境探测前，**MUST NOT** 执行安装/更新/卸载。
2. **命令必须真实执行**：**MUST NOT** 凭记忆声称「已安装」；每条命令 **MUST** 记录 `exit code` 与关键 stdout/stderr（末 20 行足够）。
3. **非 0 即停**：任一步 `exit code ≠ 0` 时 **MUST STOP**，向用户报告失败步骤与原始输出；**MUST NOT** 静默跳过进入下一步（除非下文 Scenario 明确写「可继续」）。
4. **单路径串行**：**MUST NOT** 并行跑 install 与 update；**MUST NOT** 在用户未确认时 `--force` 覆盖。
5. **占位符**：`<TARGET_DIR>` = 用户**实际开发**的业务仓绝对路径（**MUST** 与 Cursor/CodeBuddy **工作区根目录**一致，含 `openspec/` 或即将创建）；`<TOOL>` = `cursor`、`codebuddy` 或 `all`（**MUST NOT** 填 `hermes`；非 Cursor/CodeBuddy **MUST STOP**）。
6. **CLI 与 pack 两层版本**：全局 `hermes-spec version` 的 `packVersion` = **CLI 自带内核版本**；目标仓 `.hermes-spec/install-record.yaml` 的 `packVersion` = **该目录已安装的 pack 版本**。二者可能不一致；升级时 **先升 CLI，再 update/install 目标仓**。
7. **汇报格式**（每阶段结束 MUST 输出）：

```text
阶段: <名称>
命令: <完整命令>
exit: <0|1|2>
摘要: <一行结论>
下一步: <继续|S-?|STOP>
```

---

## 1. Phase 0 — 环境探测（统一入口，必须先做）

按序执行；**任一步失败则 STOP**（除非标注「仅 warning」）。

### TODO-0A  Node

```powershell
node -v
```

| exit | 输出 | 判定 |
|------|------|------|
| 0 | `v20.19.0` 或更高 | ✅ 继续 |
| 0 | 低于 `v20.19.0` | ❌ STOP → 提示用户升级 Node |
| ≠0 | — | ❌ STOP → 提示安装 Node 20.19.0+ |

### TODO-0B  Hermes CLI 是否已装

```powershell
hermes-spec version
```

| exit | 判定 |
|------|------|
| 0 | 已装 → 记录 `packVersion`，继续 TODO-0C |
| ≠0 | 未装 → **路由 S-A（全局安装 CLI）**，装完再回到 TODO-0B |

### TODO-0C  OpenSpec CLI

```powershell
openspec --version
```

| exit | 判定 |
|------|------|
| 0 | 继续 TODO-0D |
| ≠0 | **路由 S-B（补装 OpenSpec）**，完成后回到 TODO-0C |

### TODO-0D  目标仓状态

```powershell
Test-Path "<TARGET_DIR>\openspec\config.yaml"
Test-Path "<TARGET_DIR>\.hermes-spec\install-record.yaml"
```

| config.yaml | install-record | 路由 |
|-------------|----------------|------|
| 否 | 否 | **S-C 全新接入**（`init`） |
| 是 | 否 | **S-C 棕地升级**（`install`，见 TODO-C2b） |
| 是 | 是 | **S-D 更新** 或 **S-E 卸载**（按用户意图） |

**Monorepo / 多子仓陷阱（MUST 向用户确认）**：若用户给的 `<TARGET_DIR>` 是**父目录**，而 Hermes/OpenSpec 实际装在子目录（如 `...\parent\gbes-server\openspec`），**MUST NOT** 对父目录 init/update；**MUST** 改 `--target` 为子仓绝对路径，或请用户确认工作区根目录。

**探测辅助**（父目录疑似误选时 MUST 跑）：

```powershell
Get-ChildItem "<TARGET_DIR>" -Directory | ForEach-Object {
  $c = Join-Path $_.FullName "openspec\config.yaml"
  if (Test-Path $c) { "子仓候选: $($_.FullName)" }
}
```

### TODO-0E  IDE 工具（仅 init 需要；先于任何 init/install/update）

| 用户 IDE | `<TOOL>` | Agent 动作 |
|----------|----------|------------|
| Cursor | `cursor` | 必填进命令 |
| CodeBuddy | `codebuddy` | 必填进命令 |
| 两侧都要 | `all` | 必填进命令 |
| Qoder / Trae / Windsurf / 其它未适配产品 | — | **MUST STOP**。说明 Hermes-Spec 当前只验证 Cursor 与 CodeBuddy，**不会**往该 IDE 目录写技能；**MUST NOT** 把 `<TOOL>` 填成 `cursor` 继续装 |
| IDE 不明 | — | **MUST 询问**「Cursor / CodeBuddy / 都不是」。都不是 → 同上 STOP |

空仓 / 足迹为空时 **MUST** 显式 `--tool cursor|codebuddy|all`。**MUST NOT** 默认 `cursor`。

---

## 2. 场景路由

| ID | 场景 | 触发条件 |
|----|------|----------|
| **S-A** | 全局安装 hermes-spec CLI | TODO-0B 失败 |
| **S-B** | 补装 OpenSpec CLI | TODO-0C 失败 |
| **S-C** | 目标仓首次接入 / 棕地补装 | 无 install-record |
| **S-D** | 更新已安装 Hermes | 有 install-record，用户要更新 |
| **S-E** | 卸载 Hermes（保留 OpenSpec 基座） | 有 install-record，用户要卸载 |
| **S-F** | doctor 诊断 / 安装后验收 | 任意阶段后用户要检查 |
| **S-G** | 故障：install exit 1 仅 guidance | install/init 失败但提示 guidance |
| **S-H** | doctor warning 逐项修复 | exit 0 但 warnings/guidance 需处理 |

---

## S-A  全局安装 hermes-spec CLI

**WHEN** `hermes-spec version` 不存在。

### TODO-A1  安装

```powershell
npm install -g hermes-spec@latest
```

- exit 0 → TODO-0B 重跑  
- exit ≠ 0 → STOP（检查 npm registry / 权限）

> 用户指定版本时用 `@0.6.1` 等；`@latest` 与 CLI `packVersion` 对齐即可。

---

## S-B  补装 OpenSpec CLI

**WHEN** `openspec --version` 不存在。

### TODO-B1

```powershell
hermes-spec setup --yes
```

- exit 0 → TODO-0C 重跑  
- exit ≠ 0 → STOP，向用户展示 `guidance[]` 原文

---

## S-C  目标仓首次接入 / 棕地补装

**WHEN** 目标仓无 `.hermes-spec/install-record.yaml`。

### TODO-C1  一条龙 init（**仅**无 `openspec/config.yaml` 时）

```powershell
hermes-spec init --yes --target "<TARGET_DIR>" --tool <TOOL>
```

- exit **0** → 进入 TODO-C3  
- exit **1** → **路由 S-G**（读 stderr/guidance，**MUST NOT** 声称成功）  
- exit **2** → STOP（Node/OpenSpec 版本不满足）

### TODO-C2  分步（无 config 或用户明确要求）

```powershell
hermes-spec setup --yes
hermes-spec init --target "<TARGET_DIR>" --tool <TOOL>
```

每步 exit 0 才继续下一步；否则 STOP。

- **MUST** 用 Hermes 单数 `--tool`（`<TOOL>` = `cursor` / `codebuddy` / `all`）。`all` 由 `hermes-spec init` **对内**转译为官方 `--tools cursor,codebuddy`。
- **MUST NOT** 自行执行 `openspec init --tools`；**MUST NOT** 把 Hermes `all` 写成官方 `--tools all`。

### TODO-C2b  棕地升级（**已有** `openspec/config.yaml`，无 install-record）

**WHEN** 旧版 Hermes 已手工接入或 0.3.x 时代未写 install-record（常见于长期业务仓）。

```powershell
npm install -g hermes-spec@latest
hermes-spec install --yes --target "<TARGET_DIR>" --tool <TOOL>
```

- **MUST NOT** 对已有完整 OpenSpec 业务仓盲目 `init`（会在错误目录新建第二套 Hermes）  
- exit 0 → TODO-C3；用户资产（`openspec/changes`、`openspec/specs`、自定义 rules）由安装器跳过，**MUST** 在摘要中说明 `skippedUserAssets`

### TODO-C3  安装后验收

```powershell
hermes-spec doctor --target "<TARGET_DIR>"
```

- exit **0** → 向用户报告：install-record 路径、packVersion、openspecInitRan（若有 JSON 输出则解析）  
- exit **0** 但 JSON 含 `warnings` → **仍算成功**，**MUST** 逐条原文转述 warnings（含「未适配 IDE，Hermes 不会写入该目录」）；可路由 **S-H** 逐项修复；**MUST NOT** 省略警告  
- exit ≠ 0 → STOP

### TODO-C3b  RAG（Hermes doctor 用）

**WHEN** doctor 报 `RAG provider type and name are not configured` 或 `kbConfigured: false`。

编辑 `<TARGET_DIR>/.hermes-spec/rag.yaml`：

```yaml
version: 1
providerType: ""   # Skill | MCP
providerName: ""   # Skill 名或 MCP 命名空间
```

填写合法 `providerType` 与非空 `providerName` 才算已配置。历史非空 `endpoint` **不得**判为已就绪（doctor 会警告 `Legacy RAG endpoint is ignored and does not mean configured`）。
**MUST NOT** 把某一提供者文档里的服务地址当作 Hermes 已配置。

### TODO-C4  关键产物抽检（MUST grep，exit 0 即可）

在 `<TARGET_DIR>` 下确认存在（路径缺一 → 报告异常，**MUST NOT** 标 COMPLETE）。**只验所选 IDE**：`<TOOL>=cursor` 只抽 `.cursor/...`；`codebuddy` 只抽 `.codebuddy/...`；`all` 两侧都验。**MUST NOT** 因对侧目录不存在判失败。

- `.cursor/skills/openspec-hermes-apply/SKILL.md`（仅当所选含 Cursor）或 `.codebuddy/skills/...`（仅当所选含 CodeBuddy）
- `openspec/hermes/agents/review/acceptance-verifier.md`
- `openspec/config.yaml`

---

## S-D  更新 Hermes

**WHEN** 存在 `install-record.yaml` 且用户要更新。

### TODO-D0  先升全局 CLI（MUST）

```powershell
npm install -g hermes-spec@latest
hermes-spec version
```

- 记录 CLI `packVersion`；低于用户目标版本 → STOP 或改用 `@x.y.z`

### TODO-D1

```powershell
hermes-spec update --target "<TARGET_DIR>"
```

| exit | 处置 |
|------|------|
| 0 | TODO-D2（JSON 可能含 `warnings`，如提示自行 `openspec update`） |
| 1 | STOP（常见：无 install-record → 改 **S-C2b install**、网络/registry） |

### TODO-D2

```powershell
hermes-spec doctor --target "<TARGET_DIR>"
```

- 对比 install-record 中 `packVersion` 是否已变为目标版本  
- exit 0 → 向用户报告新版本号；仍有 warnings → **S-H**

---

## S-E  卸载 Hermes

**WHEN** 用户明确要求卸载，且存在 install-record。

### TODO-E1  脏工作区检查

```powershell
cd "<TARGET_DIR>"; git status --short
```

- 有未提交修改 → **MUST** 询问用户是否 `--yes` 强制；未确认 **MUST NOT** 继续

### TODO-E2

```powershell
hermes-spec uninstall --target "<TARGET_DIR>"
```

- 无 `--yes` 且 CLI 拒绝 → 向用户说明原因  
- exit 0 → 确认 `.hermes-spec/install-record.yaml` 已删除；**OpenSpec 基座文件保留**（符合设计）

---

## S-F  doctor 诊断

```powershell
hermes-spec doctor --target "<TARGET_DIR>"
```

**MUST** 向用户转述（若有）：

- `md2wordReady` / `guidance[]` / `kbConfigured`  
- OpenSpec / Node 版本不满足项  

doctor exit 0 + warnings = **通过但需知情**；exit ≠ 0 = STOP。用户要求「把 warning 也修掉」→ **S-H**。

---

## S-H  doctor warning / guidance 逐项修复

**WHEN** `doctor` exit 0，但 `warnings[]` 或 `guidance[]` 非空，且用户要求处理。

按 JSON 原文匹配下表；**每修一项后 MUST 重跑** `hermes-spec doctor --target "<TARGET_DIR>"`。

| warning / guidance 关键词 | 原因 | 修复（Agent 可执行） |
|---------------------------|------|----------------------|
| `RAG provider type and name are not configured` / `kbConfigured: false` | Hermes 持久记忆未配 | 编辑 `.hermes-spec/rag.yaml` 的 `providerType`（Skill\|MCP）与 `providerName`（见 TODO-C3b） |
| `Legacy RAG endpoint is ignored and does not mean configured` | 历史地址字段残留 | 填类型+名称后可手清旧 `endpoint`；旧地址不算已配置 |
| `config skeleton incomplete: rules.proposal` | `openspec/config.yaml` 中 `rules.proposal` 等键缩进错误或缺失 | 检查 `rules:` 下各键须 **2 空格缩进**；或 `hermes-spec install --target "<TARGET_DIR>"` 补缺失键（不覆盖已有） |
| `PRD 目录命名不一致` | `PRD.md` 文首「主 JIRA」含反引号/加粗/括号，校验器只取第一个 token | 改为 `> **主 JIRA**：HFJF-12345`（纯 JIRA 号）；说明放下一行 blockquote |
| `Python runtime is not available` | Windows 上旧 CLI（≤0.6.0）Python 探测 bug | `npm install -g hermes-spec@0.6.1+` 后重跑 doctor；本机需 `py -3` 或 `python` 可用 |
| `liteparse-doc` legacy rule | 与 `md2word-doc` 冲突 | 删除 `.cursor/rules/liteparse-doc.mdc` 与 `.codebuddy/rules/liteparse-doc.mdc`（若存在） |
| `openspec update` yourself | 官方 OpenSpec 生成物需人工刷新 | 在目标仓执行 `openspec update`（安装器 **不会** 代跑） |
| `CodeGraph` | 目标仓未 init 索引 | 在 `<TARGET_DIR>` 执行 `codegraph init` |

**MUST NOT** 把某一提供者文档里的服务地址当作 Hermes 已配置。不得把填写 endpoint 作为就绪路径。

---

## S-G  install/init 失败仅 guidance

**WHEN** install 或 init exit 1，输出含 guidance 文案。

1. **MUST** 原文列出 guidance  
2. **MUST NOT** 将 exit 1 报告为「安装成功」  
3. 按 guidance 修复前置（常见：先 `hermes-spec setup --yes` 或补 `openspec init`）  
4. 修复后 **从 TODO-0 重跑**，**MUST NOT** 假设已修复

---

## 3. 特殊参数（仅用户明确要求时使用）

| 参数 | 含义 | 风险 |
|------|------|------|
| `--skip-openspec-init` | install 跳过 OpenSpec 前置检测 | 无 config 时会 exit 1 |
| `--core-only` | 仅内核，不装 skills/commands | 高级场景 |
| `--tool codebuddy` | 仅写入 `.codebuddy` 受管树 | `<TOOL>` 必须是官方 id |
| `--tool all` | 同时写入 `.cursor` 与 `.codebuddy` | 须显式 `all` |

---

## 4. 禁止事项（防弱模型幻觉）

| 禁止 | 原因 |
|------|------|
| 未跑 doctor 就声称「安装完成」 | 伪 PASS |
| 把 doctor 的 warning 当不存在 | KB/md2word 等需用户知情 |
| 对已有 OpenSpec 的业务仓盲目 `init` | 易在 monorepo 父目录误装第二套 Hermes |
| 把父目录当 `<TARGET_DIR>` 而 IDE 打开的是子仓 | 路由错误，update 找不到 install-record |
| 用 rag-cli 技能 URL 代替 `.hermes-spec/rag.yaml` | doctor 仍报 KB 未配置 |
| 在业务仓执行 `npm publish` | 与 Hermes CLI 无关 |
| 修改 `openspec/config.yaml` 中 `schema` 键为随意值 | 异构仓 D10 纪律 |
| 用 `openspec init --force` 代替 Hermes 流程 | BR-23；安装器不代呼 force |

---

## 5. 模型稳定性自检（Agent 自测，完成任一场景后 MUST 核对）

| # | 检查 | 通过标准 |
|---|------|----------|
| 1 | 是否先跑 Phase 0 再分支 | 是 |
| 2 | 每条命令是否有 exit 记录 | 是 |
| 3 | 是否存在 exit≠0 仍标成功 | 否 |
| 4 | `<TARGET_DIR>` 是否为用户指定绝对路径 | 是 |
| 5 | 失败时是否给出**可执行**下一步（命令级） | 是 |
| 6 | 是否把 Scenario 外步骤混入（如擅自 git commit） | 否 |
| 7 | `<TARGET_DIR>` 是否与 IDE 工作区根一致 | 是 |
| 8 | 升级是否先升全局 CLI 再 update 目标仓 | 是（S-D） |

**稳定性设计说明**（给维护者）：本文采用「Phase0 探测 → 单场景 TODO 串行 → exit 码硬停」，避免「MAY/可选/视情况」；弱模型只需按表分支，强模型亦不能跳过探测。若某模型仍跳步，优先收紧为 **MUST STOP** 而非增加自由发挥段落。

---

## 6. 用户一句话 → 场景映射（快速路由）

| 用户说 | 路由 |
|--------|------|
| 「帮我在这个仓库装 Hermes」 | Phase0 确认 `<TARGET_DIR>` → 无 config：**S-C1 init**；有 config 无 record：**S-C2b install** |
| 「升级 Hermes」 | Phase0 → **S-A/D0 升 CLI** → **S-D**（无 record 则 **S-C2b**） |
| 「卸载 Hermes」 | Phase0 → S-E |
| 「检查一下 Hermes 装好了没」 | S-F（有 warning 且用户要修 → S-H） |
| 「openspec 命令不存在」 | S-B |
| 「hermes-spec 命令不存在」 | S-A |

---

## 7. 参考（人类可读）

- 内核仓 `README.md`  
- 目标仓安装后：`assets/docs/hermes-spec-agent-flow.md`（init 后位于 `.cursor` 或 `.codebuddy` 镜像路径）
