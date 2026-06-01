---
name: team-distill
description: 团队级 PRD 蒸馏 — 跨多个仓库（前端/BFF/后端）的 PRD 蒸馏，从团队 reference 原样副本生成 team-plan 和各仓库 sub-plan。适用于用户调用 /team-distill，且已有团队 knowledge base 时。
---

# team-distill

通过 `/team-distill <PRD 文件或需求文本>` 触发。

**前置条件**：`project-profile.yaml` 存在且 `layer: "team-common"`，或 `references/` 目录存在且有子目录。

## 与单仓模式的差异

| 方面 | 单仓 `/prd-distill` | 团队 `/team-distill`（fan-out/fan-in） |
|------|---------------------|---------------------------------------|
| 编排模型 | 单 agent 顺序 11 步 | 主 agent 跑 Step 1-3 + 3.5 → 涉及仓 fan-out subagent → 主 agent 聚合（Step 7.5/7.6/8/9-11） |
| 源码扫描 | rg/glob + reference | 主 agent 禁止 rg/glob；**subagent 在 `repos/{repo}/` 内可正常 rg/glob** |
| Step 3.5 涉及仓识别（新增） | 不适用 | 主 agent 用 PRD 关键词 × 各仓 01-codebase + 04-routing-playbooks 匹配，输出 `involved_repos[]` |
| Step 4-7 | 主 agent 直接跑 | **fan-out**：主 agent 用 Agent 工具并行 dispatch subagent，每个 subagent 在自己仓内跑 Step 4-7（参见 prd-distill/workflow.md "single-repo subagent 模式"） |
| Step 4.5 Context Pack | 单仓 | 从 `references/{repo}/index/` 加载多仓 index（`context-pack.py --team-references`） |
| Contract Delta | 单仓视角 | subagent 出本仓 contract-delta；**主 agent Step 7.5 跨仓对齐**，产物 `context/cross-align.yaml` |
| Step 7.6 Report 聚合 | — | 主 agent 把各仓 report 摘要合并到团队 report.md §9.{repo}，§9.5 由 cross-align 渲染 |
| Plan | 1 份 plan.md | 主 agent 出 1 份 team-plan.md + N 份 plans/plan-{repo}.md（基于 per-repo report 生成） |
| 输出布局 | `<distill>/{report.md, plan.md, context/}` | `<distill>/{per-repo/{repo}/, report.md, team-plan.md, plans/, context/}` |

成员仓列表来自 `project-profile.yaml` 的 `team_repos[]`，每仓需配置 `source_path` 与 `submodule: true`。

## 核心职责

与单仓模式相同（详见 `skills/prd-distill/SKILL.md`），但面向多仓库：
1. 从各仓库 reference 副本获取跨仓实体、契约、规则。
2. 自动识别 PRD 涉及哪些仓库及角色（producer/consumer/middleware）。
3. 生成团队级总计划 + 各仓库独立 sub-plan。

## 触发条件

- 用户调用 `/team-distill`。
- `project-profile.yaml` 含 `layer: "team-common"` 或 `references/` 目录存在。

不触发：无团队 knowledge base、单仓项目。

## 输入

同单仓模式，额外：
- `project-profile.yaml`：团队配置（`team_repos[]`）。
- `references/{repo}/`：各成员仓库的 reference 原样副本（01-05 YAML + index）。

## 输出结构

```text
_prd-tools/distill/<slug>/
├── _ingest/                          # 主 agent 一次落地（PRD/document.md）
├── per-repo/                         # 新增：subagent 全量产物（每仓一目录）
│   └── {repo}/
│       ├── report.md
│       ├── context/
│       │   ├── layer-impact.yaml
│       │   ├── contract-delta.yaml
│       │   └── ...
│       ├── evidence/
│       └── _failure.json             # 仅失败仓产生
├── report.md                         # 团队级报告（§9 分 5+N 子节）
├── team-plan.md                      # 团队级开发计划总览
├── plans/
│   ├── plan-{repo1}.md
│   └── plan-{repo2}.md
└── context/
    ├── requirement-ir.yaml           # 主 agent 一次生成，广播给 subagent
    ├── layer-impact.yaml             # 主 agent 聚合（4 层来自各仓）
    ├── contract-delta.yaml           # 主 agent 聚合
    ├── cross-align.yaml              # 新增：跨仓对齐结论
    └── ...
```

## 参考文件

| 文件 | 何时读取 |
|---|---|
| `workflow.md`（本文件） | 执行团队蒸馏时 |
| `skills/prd-distill/workflow.md` | 通用步骤详情 |
| `skills/prd-distill/references/output-contracts.md` | 输出格式定义 |
| `skills/prd-distill/references/layer-adapters.md` | 能力面定义 |
