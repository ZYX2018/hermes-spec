# Hermes-Spec · 公开安装指引

本仓库是 **Hermes-Spec AI 安装文档** 的公开镜像，供社区阅读、试用与反馈。

Hermes-Spec 是一套面向 **OpenSpec 工作区** 的内核包与安装器：在 Cursor、CodeBuddy 等 IDE 中，为 AI Agent 提供 propose / apply / 评审 / 验收等工程化工作流技能与门禁脚本。

> **说明**：完整内核代码与 npm 发行包不在本仓库；此处仅托管 **AI 可执行的安装 / 更新 / 卸载指引**。

---

## 快速开始

| 角色 | 做什么 |
|------|--------|
| **人类开发者** | 把下面文档交给 IDE 里的 AI，让它在你的**业务代码仓**完成安装 |
| **AI Agent** | 直接阅读并严格按文档执行（含环境探测、分场景路由、exit code 汇报） |

👉 **[HERMES-SPEC-AI-INSTALL.md](./HERMES-SPEC-AI-INSTALL.md)** — 主文档（当前对齐 `hermes-spec@0.6.1+`）

**最低环境**：Node ≥ 20.19.0 · OpenSpec CLI effectiveMin ≥ 1.6.0

```powershell
# 全局安装 CLI（示例）
npm install -g hermes-spec

# 在目标业务仓一条龙接入（由 AI 按文档补全 --target / --tools）
hermes-spec init --yes --target <你的业务仓绝对路径>
hermes-spec doctor --target <你的业务仓绝对路径>
```

---

## 文档里有什么

- **AI 执行契约**：先探测后分支、命令必须真实执行、非 0 即停
- **Phase 0 环境探测**：Node / Hermes CLI / OpenSpec / 目标仓状态
- **分场景路由**：全新接入、棕地升级、更新、卸载
- **Monorepo 陷阱**：工作区根目录与 `--target` 对齐说明
- **doctor 解读**：RAG / md2word 等可选能力的状态判定

---

## 反馈与贡献

欢迎通过 **[Issues](https://github.com/ZYX2018/hermes-spec/issues)** 提交：

- 安装步骤在某 IDE / 模型下走不通
- 文档表述歧义或缺少 Scenario
- 环境探测结论与实际情况不符

请尽量附上：**操作系统、Node 版本、IDE、完整命令与 exit code / 输出末 20 行**。

---

## 仓库关系

| 仓库 | 用途 |
|------|------|
| **本仓库（GitHub · 公开）** | AI 安装指引 · 社区反馈 |
| **内核仓（私有/内网）** | npm 包、skills、OpenSpec 规范与 CI |

文档更新会定期从内核仓同步至本仓库；以 [`HERMES-SPEC-AI-INSTALL.md`](./HERMES-SPEC-AI-INSTALL.md) 文内版本说明为准。
