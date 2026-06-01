# 跨仓蒸馏升级：团队仓内含源码 + Fan-out/Fan-in 编排

**日期**：2026-06-01
**作者**：Neal + Claude (Opus 4.7)
**范围**：`team-distill`（核心改动）+ `team-reference`（轻量改动）
**目标**：把现有「主 agent 顺序读 reference」的团队蒸馏，升级为「团队仓内含成员仓源码 + 主 agent 编排 subagent 并行蒸馏 + 主 agent 跨仓聚合」的两段式架构。

---

## 1. 背景与目标

### 1.1 现状

- `/team-reference` Mode T：把各成员仓的 `_prd-tools/reference/` 原样复制到团队仓 `references/{repo}/`，不聚合
- `/team-distill`：主 agent 顺序读所有 `references/{repo}/`，**禁止 rg/glob**（团队仓没有源码），生成 1 份 team-plan + N 份 sub-plan
- 跨仓推理（contract 对齐、cross_repo_handoffs）依赖各仓 reference 中已有的 `producer/consumers[]` 字段

### 1.2 痛点

- 团队仓只有 reference，没有源码 → 无法验证 reference 是否过时、无法对模糊需求二次定位代码
- 单 agent 串行处理 N 仓 → 上下文压力大、跨仓推理与单仓细节互相干扰
- 单仓 confidence 难以提升（只有 reference，没有源码兜底）

### 1.3 目标

1. **团队仓内含源码**：用 git submodule 引入各成员仓源码到 `repos/{repo}/`
2. **fan-out/fan-in 编排**：主 agent 识别涉及仓 → 并行 dispatch subagent → 各 subagent 在自己仓的源码上跑 prd-distill Step 4-7 → 主 agent 聚合并跨仓对齐
3. **不破坏现有单仓 `/prd-distill` 与 `/reference`**：subagent 复用单仓 prd-distill 的 Step 4-7

---

## 2. 决策记录

| # | 决策 | 选定 | 备注 |
|---|------|------|------|
| 1 | 源码引入方式 | git submodule | 标准做法，HEAD 可锁，refresh 一致 |
| 2 | subagent 能力面 | 复用单仓 prd-distill，按 step 子集（Step 4-7）调用 | 不新增 skill，避免反膨胀 |
| 3 | 涉及仓识别时机 | 主 agent Step 3.5 预扫识别，只 fan-out 相关仓 | 节省 token；漏判由 quality-gate 启发式提示兜底 |
| 4 | 跨仓推理位置 | 完全由主 agent 在 Step 7.5 聚合时做 | subagent 只看自己仓，边界最清 |
| 5 | per-repo 输出落地 | 单独 `per-repo/{repo}/` 子目录，全量产物 | 可追溯、可独立重跑某仓 |
| 6 | 单 subagent 失败处理 | 标 unavailable，其余仓照常，最终 quality-gate 警告 | 不阻断、不重试 |
| 7 | 源码同步责任 | 用户手动 `git submodule update`；team-reference 只校验 HEAD 一致性 | team-reference 职责保持单一 |

---

## 3. 数据布局

### 3.1 目录结构

```text
团队仓根/
├── .gitmodules                          # 新增：submodule 配置
├── project-profile.yaml                 # team_repos[] schema 扩展
├── references/                          # 不变（team-reference 产物）
│   └── {repo}/
│       ├── 01-codebase.yaml
│       ├── 02-coding-rules.yaml
│       ├── 03-contracts.yaml
│       ├── 04-routing-playbooks.yaml
│       ├── 05-domain.yaml
│       ├── project-profile.yaml         # 含 git_head
│       └── index/...
├── repos/                                # 新增：成员仓 git submodule
│   ├── {repo-1}/                         # submodule 1
│   ├── {repo-2}/                         # submodule 2
│   └── ...
└── _prd-tools/
    └── distill/<slug>/
        ├── _ingest/
        │   └── prd.md                    # 主 agent 一次落地
        ├── per-repo/                     # 新增：subagent 全量产物
        │   └── {repo}/
        │       ├── report.md
        │       ├── context/
        │       │   ├── layer-impact.yaml
        │       │   ├── contract-delta.yaml
        │       │   ├── graph-context.yaml
        │       │   └── ...
        │       ├── evidence/
        │       └── _failure.json         # 仅失败时产生
        ├── report.md                     # 主 agent 聚合产物
        ├── team-plan.md
        ├── plans/
        │   └── plan-{repo}.md            # 主 agent 基于 per-repo report 生成
        └── context/
            ├── requirement-ir.yaml       # 主 agent 一次生成，广播给 subagent
            ├── layer-impact.yaml         # 主 agent 聚合（4 层来自各仓）
            ├── contract-delta.yaml       # 主 agent 聚合
            ├── cross-align.yaml          # 新增：跨仓对齐结论
            └── ...
```

### 3.2 project-profile.yaml schema 扩展

现有 schema（`plugins/reference/skills/reference/templates/project-profile.yaml`）：

```yaml
team_repos:
  - repo: "dive-bff"
    local_path: "../dive-bff"               # 用于 /team-reference 定位成员仓拷贝 reference
    layer: "bff"                            # frontend | bff | backend
```

升级后：

```yaml
team_repos:
  - repo: "dive-bff"
    local_path: "../dive-bff"               # 不变（/team-reference 仍按此复制 reference）
    source_path: "repos/dive-bff"           # 新增：相对团队仓根；/team-distill 的 subagent CWD
    submodule: true                         # 新增：标记是否托管为 git submodule
    layer: "bff"
  - repo: "dive-fe"
    local_path: "../dive-fe"
    source_path: "repos/dive-fe"
    submodule: true
    layer: "frontend"
```

字段语义：
- `local_path`：用户机上各成员仓的工作副本路径，用于 /team-reference 时复制 reference（保持原义）
- `source_path`：成员仓源码在团队仓内的相对路径，用于 /team-distill 时 subagent 的 CWD 定位
- `submodule`：标记是否为 git submodule（影响 HEAD 校验和文档提示语）

### 3.3 .gitignore 与 git 行为

- `repos/` **不**进 .gitignore；submodule 引用本身要进 git
- `repos/{repo}/_prd-tools/` 由各成员仓自己管理，团队仓不关心
- `references/` 维持现状（由 /team-reference 维护）

---

## 4. team-reference 改动（轻量）

[plugins/reference/skills/team-reference/workflow.md](../../../plugins/reference/skills/team-reference/workflow.md) Mode T 流程：

| 步骤 | 现状 | 升级后 |
|------|------|--------|
| 1. 读取配置 | `team_repos[]` | 同（schema 已扩展，但 team-reference 不消费 source_path） |
| 2. 逐个收集 | 复制 `local_path/_prd-tools/reference/` → `references/{repo}/` | 不变 |
| 3. **HEAD 校验**（新增） | — | 若 `repos/{repo}/` 存在（submodule 已 init），对比 `git -C repos/{repo} rev-parse HEAD` 与 `references/{repo}/project-profile.yaml` 的 `git_head`，**不一致只发警告**，不阻断 |
| 4. 输出摘要 | 收集成功/跳过 | 增加 `head_mismatch[]` 字段 |

**team-reference 不主动跑 `git submodule update`**（用户决策）。

**改动量**：workflow.md 增加约 20 行（HEAD 校验描述 + 摘要字段）。无新建脚本。

---

## 5. team-distill 编排架构（核心）

### 5.1 全流程

```text
┌─────────────────────────────────────────────────────────────┐
│ 主 agent                                                     │
│                                                              │
│ Step 1  PRD Ingestion       ──┐                              │
│ Step 2  Evidence              │ 单仓无关，主 agent 一次完成   │
│ Step 3  Requirement IR      ──┘ 产物广播给 subagent           │
│                                                              │
│ Step 3.5 涉及仓识别（新增）                                   │
│   读 references/{repo}/01-codebase + 04-routing-playbooks      │
│   PRD 关键词 × 模块名/路由 匹配 → involved_repos[]            │
│   每仓附 hint: 角色（producer/consumer/middleware）           │
│                                                              │
│   ↓ fan-out（仅 involved_repos）                             │
└──────────────────┬──────────────┬──────────────────┬────────┘
                   ▼              ▼                  ▼
            ┌──────────┐    ┌──────────┐      ┌──────────┐
            │subagent A│    │subagent B│ ...  │subagent N│
            │ CWD =    │    │ CWD =    │      │ CWD =    │
            │ repos/A  │    │ repos/B  │      │ repos/N  │
            │          │    │          │      │          │
            │ Step 4-7 │    │ Step 4-7 │      │ Step 4-7 │
            │ (prd-    │    │ (prd-    │      │ (prd-    │
            │  distill)│    │  distill)│      │  distill)│
            │          │    │          │      │          │
            │ → per-   │    │ → per-   │      │ → per-   │
            │   repo/A │    │   repo/B │      │   repo/N │
            └────┬─────┘    └─────┬────┘      └─────┬────┘
                 │                │                 │
                 └────────────────┴─────────────────┘
                                  │ barrier（等齐所有仓）
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│ 主 agent 聚合阶段                                             │
│                                                              │
│ Step 7.5 跨仓对齐（新增）                                     │
│   读所有 per-repo/{repo}/context/contract-delta.yaml          │
│   横向比对：producer ↔ consumer 闭环、handoff、owner          │
│   产出 context/cross-align.yaml + report.md §9.5             │
│                                                              │
│ Step 7.6 Report 聚合                                         │
│   合并各仓 report.md 摘要 → 团队 report.md §9.{repo}          │
│   §9.5 = cross-align 结论                                    │
│                                                              │
│ Step 8  Plan 生成                                            │
│   team-plan.md（跨仓时序、依赖、sub-plan 索引）               │
│   plans/plan-{repo}.md（主 agent 基于 per-repo report 生成）  │
│                                                              │
│ Step 9-11 Readiness / Backflow / Quality Gate                │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 关键性质

- **subagent 的 CWD = `repos/{repo}/`**：可以正常 rg/glob 真源码 — **解除现有团队模式"禁止 rg/glob"约束**
- **subagent 边界**：只看自己仓的 source + reference，不读其他仓的 reference/source
- **跨仓推理完全在主 agent**：subagent 不做跨仓判断，避免不一致结论
- **Step 8 plan 主 agent 出**：保证 sub-plan 风格一致 + 跨仓时序对齐
- **fan-out 后 barrier**：跨仓对齐必须看齐所有 subagent 产物，不做流水线

### 5.3 涉及仓识别（Step 3.5）算法

输入：requirement-ir.yaml + 所有 `references/{repo}/{01-codebase, 04-routing-playbooks}.yaml`

规则（启发式，确定性）：

1. 对每个 REQ 提取关键词（实体名、动作名、路由片段）
2. 与各仓 `01-codebase.yaml` 的 modules/entities 名称匹配 → 命中即标记该仓相关
3. 与各仓 `04-routing-playbooks.yaml` 的 routes/handoffs 匹配 → 命中即标记该仓相关
4. 角色推断：依据 `03-contracts.yaml` 的 producer/consumers，决定该仓是 producer/consumer/middleware
5. 漏判兜底：Step 7.5 cross-align 阶段如果发现 `consumer_orphan`，在 §9.5 提示"疑似漏 fan-out 仓"，让用户决定是否补跑

输出：

```yaml
involved_repos:
  - repo: dive-bff
    role: middleware
    matched_via: ["module:order", "route:/api/v1/order"]
    confidence: high
  - repo: dive-fe
    role: consumer
    matched_via: ["entity:Order"]
    confidence: medium
not_involved_repos: [dive-be-legacy]   # 主 agent 显式排除清单（用于审查）
```

---

## 6. subagent 边界与契约

### 6.1 调用方式

主 agent 在 Step 3.5 之后，对每个 `involved_repos[i]` 用 `Agent` 工具并行 dispatch（subagent_type=`general-purpose`）。

### 6.2 subagent prompt 模板

主 agent 构造，传给每个 subagent：

```
你正在团队级 PRD 蒸馏的 fan-out 阶段，负责仓库：{repo}。

工作目录（CWD）：repos/{repo}/    ← 真实源码，可用 rg/glob/Read
参考资料：references/{repo}/      ← 该仓 reference（与源码 HEAD 应对齐）

输入产物（团队仓的相对路径，主 agent 已经准备好）：
- _prd-tools/distill/{slug}/_ingest/prd.md
- _prd-tools/distill/{slug}/context/requirement-ir.yaml
- 你被识别为：role={producer|consumer|middleware}（hint，可以推翻）

任务：调用 superpowers:prd-distill skill，按 "single-repo subagent 模式"
跑 Step 4-7（Code Search / Layer Impact / Contract Delta / Report），
**禁止跑 Step 1-3、Step 8-11**。

输出落到（团队仓相对路径）：
_prd-tools/distill/{slug}/per-repo/{repo}/
  ├── report.md
  ├── context/
  │   ├── layer-impact.yaml      # 4 层填充，code_anchors 指向 repos/{repo}/...
  │   ├── contract-delta.yaml    # 本仓视角的 producer/consumer 边界
  │   ├── graph-context.yaml
  │   └── ...
  ├── evidence/
  └── _ingest/                   # 软链或拷贝团队 _ingest（保持单仓 prd-distill 路径不变）

成功标准：
- 4 层 IMP 完整填充（外部层可标 confidence: low）
- contract-delta 的 producer/consumers 字段从本仓 03-contracts 推
- 不读其他仓的 reference 或 source

失败时：返回 stderr 和已生成的部分产物路径，写 _failure.json，不要重试。
```

### 6.3 输入输出契约

| 项 | 内容 |
|----|------|
| 输入路径 | 团队仓 PRD + requirement-ir 路径（团队仓根的相对路径） |
| 输入提示 | 仓名 + 角色 hint（producer/consumer/middleware） |
| 工作目录 | subagent 启动后 cd 到 `repos/{repo}/` |
| **禁止** | 读 `references/{other_repo}/`、读 `repos/{other_repo}/`、生成 plan、跑 readiness/gate |
| 输出 | `_prd-tools/distill/{slug}/per-repo/{repo}/` 下完整产物 |
| 返回值 | JSON：`{repo, status: ok\|partial\|failed, output_dir, summary, errors[]}` |

### 6.4 prd-distill workflow.md 改动

在 [plugins/prd-distill/skills/prd-distill/workflow.md](../../../plugins/prd-distill/skills/prd-distill/workflow.md) 末尾加约 30 行 "single-repo subagent 模式" 小节：

> 当被 team-distill 主 agent 以 subagent 形式调用时（提示里包含 `single-repo subagent 模式`），跳过 Step 1-3 和 Step 8-11，只跑 Step 4-7，输出路径改写到 `_prd-tools/distill/{slug}/per-repo/{repo}/`。其余流程不变。

**为什么不新建 sub-distill skill**：CLAUDE.md 反膨胀规则禁止"为了复用的新抽象 / 并行规范文件"。在 prd-distill workflow.md 中加一个分支说明，避免双源真相。

---

## 7. 主 agent 聚合：Step 7.5 + 7.6 + 8

### 7.1 Step 7.5 跨仓对齐

**输入**：所有 `per-repo/{repo}/context/contract-delta.yaml`

**算法**（确定性）：

1. **Endpoint 索引**：把每仓 contract-delta 的 endpoint 抽到 `(method, path)` 键，记录 `repo + role(producer|consumer)`
2. **闭环检查**：对每个 endpoint：
   - 同 key 同时有 producer 和 consumer → `aligned`
   - 只有 producer，无 consumer → `producer_orphan`（可能正常）
   - 只有 consumer，无 producer → `consumer_orphan`（**风险**：调用了不存在的下游）
   - 多个 producer → `producer_conflict`（**风险**：契约重复声明）
3. **字段一致性**：同 key producer 与 consumer 的 request/response schema 字段对比，diff 写入 `field_drift[]`
4. **Owner 缺位**：endpoint 有 producer 但 03-contracts 中 `owner` 为空 → `owner_missing`

**产物**：`context/cross-align.yaml`

```yaml
version: 1
endpoints:
  - key: "POST /api/v1/order"
    status: aligned | consumer_orphan | producer_conflict | producer_orphan
    producer: { repo: dive-bff, owner: ... }
    consumers: [{ repo: dive-fe, ... }]
    field_drift: []
    risks: []
handoffs:        # 来自各仓 04-routing-playbooks 的 cross_repo_handoffs 合并去重
  - ...
unavailable_repos: [...]   # 失败仓清单，跨仓判断中作为缺口
```

### 7.2 Step 7.6 Report 聚合

`report.md` 顶层结构：

| 章节 | 来源 |
|------|------|
| §1-§8 通用章节 | 主 agent 基于 requirement-ir + 跨仓视角生成 |
| §9.1 Frontend | 摘要 `per-repo/{fe-repo}/report.md` |
| §9.2 BFF | 摘要 `per-repo/{bff-repo}/report.md` |
| §9.3 Backend | 摘要 `per-repo/{be-repo}/report.md` |
| §9.4 External | 主 agent 从各仓 layer-impact 的 external 层合并 |
| §9.5 跨层对齐风险 | 由 `cross-align.yaml` 渲染 |
| §9.{unavail-repo} | 标 `unavailable: <reason>`，confidence=low |

**摘要规则**：每个 §9.{repo} 子节列 IMP 表 + 关键契约 delta + 该仓 report §6 风险摘要，**不全文复制**——全文留在 `per-repo/{repo}/report.md`，团队 report 给出锚点链接。

### 7.3 Step 8 Plan 生成

**team-plan.md** 沿用现有 7 节结构（[team-distill/workflow.md:82-89](../../../plugins/prd-distill/skills/team-distill/workflow.md#L82-L89)），关键改动：
- "跨仓时序" 节由 `cross-align.yaml` + 各仓 report 工作量估算驱动
- "Sub-Plan 索引表" 列每个 plan 文件路径 + IMP 数 + 是否 unavailable

**plans/plan-{repo}.md** 由主 agent **基于 `per-repo/{repo}/report.md` 生成**，沿用单仓 11-section plan 模板。subagent 不出 plan 是为了：
- 风格一致（同一个主 agent 写所有 plan）
- 跨仓时序在 team-plan 决定后，sub-plan 才能引用对应的依赖 phase
- unavailable 仓也能产出 plan（占位 + 标记 blocked）

---

## 8. 失败处理与 Quality Gate

### 8.1 失败处理矩阵

| 失败类型 | 处理 | 报告体现 |
|---------|------|---------|
| `repos/{repo}/` 不存在或 submodule 未初始化 | 主 agent 跳过 fan-out，标 unavailable | §9.{repo} 写"submodule 未初始化，需 `git submodule update --init`" |
| `repos/{repo}` HEAD ≠ `references/{repo}` 的 git_head | 主 agent 警告但继续，subagent 用源码为准 | §9.{repo} 顶部加一行 `head_drift: source=<sha1> reference=<sha2>` |
| subagent 跑崩 / 超时 | 不重试 | per-repo/{repo}/ 留下部分产物 + `_failure.json`，§9.{repo} 标 unavailable |
| 涉及仓识别（Step 3.5）漏报某仓 | quality-gate 启发式校验提示 | report.md §9.5 警告区给出"疑似漏判仓清单" |

### 8.2 Quality Gate 扩展

[plugins/prd-distill/skills/prd-distill/scripts/](../../../plugins/prd-distill/skills/prd-distill/) 现有 quality-gate 在团队模式下补加检查（**改现有脚本，不新建**）：

| 新增检查 | 通过标准 |
|---------|---------|
| per-repo 完整性 | 每个 involved_repo 有 `per-repo/{repo}/report.md` 或显式 unavailable 标记 |
| cross-align 存在 | `context/cross-align.yaml` 存在且 `endpoints[]` 非空（除非 PRD 无契约改动） |
| §9 五子节齐 | report.md §9.1-§9.5 都有内容（unavailable 仓也要占位） |
| sub-plan 数量 | `plans/plan-{repo}.md` 数 = involved_repos 数（unavailable 仓也要占位 plan） |
| 漏判提示 | endpoint 在 cross-align 中 `consumer_orphan` 时，建议核查未 fan-out 的仓 |

unavailable 仓不阻断交付，但 gate 输出 `severity: warning` 计入摘要。

---

## 9. 改动文件清单（反膨胀核验）

### 9.1 修改（6 个文件）

1. [plugins/reference/skills/reference/templates/project-profile.yaml](../../../plugins/reference/skills/reference/templates/project-profile.yaml) — `team_repos[]` 加 `source_path`、`submodule` 字段
2. [plugins/reference/skills/team-reference/workflow.md](../../../plugins/reference/skills/team-reference/workflow.md) — Mode T 增加 HEAD 一致性校验步骤
3. [plugins/prd-distill/skills/team-distill/SKILL.md](../../../plugins/prd-distill/skills/team-distill/SKILL.md) — 差异表更新（4.2 Graph Context 不再"禁止 rg/glob"，改为"主 agent 禁止；subagent 在自己仓内允许"）
4. [plugins/prd-distill/skills/team-distill/workflow.md](../../../plugins/prd-distill/skills/team-distill/workflow.md) — **核心**：Step 3.5 / 7.5 / 7.6 / 8 重写，加 fan-out/fan-in 编排说明
5. [plugins/prd-distill/skills/prd-distill/workflow.md](../../../plugins/prd-distill/skills/prd-distill/workflow.md) — 加约 30 行 "single-repo subagent 模式"
6. 现有 quality-gate 脚本 — 加 §8.2 五项检查（追加到现有函数，不新建文件）

### 9.2 新增

- 团队仓的 `.gitmodules`（用户初始化 submodule 时自动产生，不算文档）
- `_prd-tools/distill/<slug>/per-repo/{repo}/` 运行时目录（不算源码）
- 1 个新 context artifact：`context/cross-align.yaml`，对应 `contracts/cross-align.yaml` schema 文件 1 个

### 9.3 反膨胀核验

- ✅ 不新建 skill（subagent 复用 prd-distill）
- ✅ 不新建 scripts（quality-gate 改现有）
- ✅ 不新建并行规范（cross-align schema 单一权威源）
- ⚠️ 新增 1 个 contracts schema — 在上限内（每个 context artifact 1 个）

---

## 10. 暂不做（YAGNI）

- **subagent 增量缓存**（基于 PRD hash + repo HEAD 跳过未变更仓）：先把基线跑通，缓存留待 Phase 2
- **subagent 并发数限制**：先全并发；超过 5 仓再考虑分批
- **subagent 自动重试**：明确不重试
- **跨仓 schema 字段语义对齐**（不只是字段名 diff，还做语义比较）：留待后续，用 LLM judge 做
- **ADR 0012 的"team/01-codebase.yaml + cross_repo_entities 索引"**：本设计不依赖，可独立演进

---

## 11. 兼容性

- 单仓 `/prd-distill` 流程不受影响（subagent 模式只是新增分支说明）
- 单仓 `/reference` 流程不受影响
- 现有团队仓（无 `repos/` 目录）：team-reference 仍能跑（HEAD 校验跳过）；team-distill 中所有 involved_repos 因 source_path 不存在而标 unavailable，主 agent 退化为"只读 reference"模式（即旧行为，每仓在 §9.{repo} 显式标注 unavailable + 提示 `git submodule update --init`）
- ADR 0012 与本设计正交：本设计只是把"团队仓内含源码"作为前提，不改 reference 收集语义
