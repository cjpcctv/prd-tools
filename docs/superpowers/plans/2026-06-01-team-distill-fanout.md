# team-distill Fan-out/Fan-in Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `/team-distill` 重构为「主 agent 编排 + subagent 并行蒸馏 + 主 agent 聚合」的两段式架构，并在团队仓内通过 git submodule 引入成员仓源码。

**Architecture:** Spec 见 [docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md](../specs/2026-06-01-team-distill-fanout-design.md)。本次改动以**修改现有文件为主，零新增 skill / 零新增脚本**，仅向 `output-contracts.md` 加 1 个 schema 章节（cross-align.yaml），向 `quality-gate.py` 追加 4 个检查函数，向各 workflow.md 加段落。

**Tech Stack:** Markdown / YAML / Python 3（quality-gate.py，约 953 行）。无 pytest 测试框架，验证以 `python -c` 断言 + 命令行调用 + grep 校验为主。

**前置假设：**
- 工作分支 `v2.0` 上进行
- 每个 Task 完成后立即 commit（CLAUDE.md 反膨胀 + 频繁提交规范）
- Commit 用 conventional prefix：`feat:`/`refactor:`/`docs:`

---

## File Structure 概览

| 操作 | 路径 | 责任 |
|------|------|------|
| 修改 | `plugins/reference/skills/reference/templates/project-profile.yaml` | `team_repos[]` 注释加 `source_path`/`submodule` 字段 |
| 修改 | `plugins/reference/skills/team-reference/workflow.md` | Mode T 加 HEAD 校验步骤 |
| 修改 | `plugins/prd-distill/skills/prd-distill/workflow.md` | 末尾追加 "single-repo subagent 模式" 小节 |
| 修改 | `plugins/prd-distill/skills/team-distill/SKILL.md` | 差异表更新（rg/glob 限制、subagent 边界） |
| 修改 | `plugins/prd-distill/skills/team-distill/workflow.md` | **核心**：重写 Step 4-8，加 Step 3.5 / 7.5 / 7.6 |
| 修改 | `plugins/prd-distill/skills/prd-distill/references/output-contracts.md` | 加 cross-align.yaml schema 章节 |
| 修改 | `scripts/quality-gate.py` | 追加 4 个检查函数 + 集成到 `run_distill_quality` |

---

## Task 1: profile-template schema 扩展

**Files:**
- Modify: `plugins/reference/skills/reference/templates/project-profile.yaml:22-26`

- [ ] **Step 1: 阅读当前 schema 注释段**

```bash
sed -n '18,30p' plugins/reference/skills/reference/templates/project-profile.yaml
```

预期输出包含：
```yaml
# team_repos:
#   - repo: "dive-bff"
#     local_path: "../dive-bff"
#     layer: "bff"
```

- [ ] **Step 2: 在 team_repos 注释里加新字段**

替换 `plugins/reference/skills/reference/templates/project-profile.yaml` 的第 22-26 行注释段为：

```yaml
# ── 团队仓库配置（仅团队仓库填写）──
# team_repos:                                            # 成员仓库列表
#   - repo: "dive-bff"                                   # 仓库名（也是 references/ 下的目录名）
#     local_path: "../dive-bff"                          # 用户机上工作副本，用于 /team-reference 复制 reference
#     source_path: "repos/dive-bff"                      # 团队仓内的 submodule 路径，用于 /team-distill subagent CWD
#     submodule: true                                    # 是否托管为 git submodule
#     layer: "bff"                                       # frontend | bff | backend
```

- [ ] **Step 3: 验证 yaml 语法仍合法**

```bash
python3 -c "import yaml; yaml.safe_load(open('plugins/reference/skills/reference/templates/project-profile.yaml'))"
```

预期：无输出（解析成功）。

- [ ] **Step 4: 验证关键字段都在注释里**

```bash
grep -n "source_path\|submodule" plugins/reference/skills/reference/templates/project-profile.yaml
```

预期输出 2 行命中。

- [ ] **Step 5: Commit**

```bash
git add plugins/reference/skills/reference/templates/project-profile.yaml
git commit -m "feat(reference): add source_path/submodule to team_repos schema"
```

---

## Task 2: team-reference Mode T 加 HEAD 校验

**Files:**
- Modify: `plugins/reference/skills/team-reference/workflow.md`

- [ ] **Step 1: 阅读当前 Mode T 步骤**

```bash
cat plugins/reference/skills/team-reference/workflow.md
```

确认 "执行步骤" 是 1-3 三步（读取配置、逐个收集、输出摘要）。

- [ ] **Step 2: 在第 2 步与第 3 步之间插入 HEAD 校验步骤**

定位文件中：

```markdown
3. **输出摘要**：哪些仓库收集成功、哪些跳过（路径不存在或 reference 不完整）
```

替换为：

```markdown
3. **HEAD 一致性校验**（团队仓含 submodule 时）：
   - 若 `<团队仓>/repos/{repo}/` 存在（submodule 已 init），运行 `git -C repos/{repo} rev-parse HEAD` 取实际 source HEAD
   - 与 `references/{repo}/project-profile.yaml` 的 `git_head` 字段比对
   - **不一致只发警告，不阻断收集**：把该仓加入 `head_mismatch[]` 摘要字段
   - 团队仓没有 `repos/` 或某仓未 init submodule 时跳过此校验
4. **输出摘要**：哪些仓库收集成功、哪些跳过（路径不存在或 reference 不完整）、哪些 HEAD 不一致（`head_mismatch[]`）
```

- [ ] **Step 3: 在 "硬约束" 节后追加补充说明**

在文件末尾 `3. **跳过不可达仓库**：...` 之后追加：

```markdown
4. **不主动 submodule update**：team-reference 不替用户跑 `git submodule update`，源码同步是用户责任；本步骤只做"提示"，不擅自动 git。
```

- [ ] **Step 4: 验证文件结构与 grep**

```bash
grep -n "HEAD 一致性校验\|head_mismatch\|不主动 submodule update" plugins/reference/skills/team-reference/workflow.md
```

预期 3 行命中。

```bash
python3 -c "
import re
content = open('plugins/reference/skills/team-reference/workflow.md').read()
# 步骤编号 1-4 连续
nums = re.findall(r'^(\d+)\. \*\*', content, re.M)
assert nums == ['1', '2', '3', '4'], f'步骤编号不连续: {nums}'
print('OK')
"
```

预期输出 `OK`。

- [ ] **Step 5: Commit**

```bash
git add plugins/reference/skills/team-reference/workflow.md
git commit -m "feat(team-reference): add HEAD consistency check between repos/ and references/"
```

---

## Task 3: prd-distill workflow.md 加 single-repo subagent 模式说明

**Files:**
- Modify: `plugins/prd-distill/skills/prd-distill/workflow.md` (末尾追加)

- [ ] **Step 1: 检查 workflow.md 末尾结构**

```bash
tail -30 plugins/prd-distill/skills/prd-distill/workflow.md
```

记录现有最后一节的标题，新内容追加在它之后（作为同级新节）。

- [ ] **Step 2: 在文件末尾追加新章节**

在 `plugins/prd-distill/skills/prd-distill/workflow.md` 末尾追加（文件末尾留一个空行后再加）：

````markdown

---

## 附：single-repo subagent 模式（被 team-distill 调用时）

当本 skill 被 `/team-distill` 主 agent 以 subagent 形式调用时（subagent 的提示中包含字面量 `single-repo subagent 模式`），按以下规则裁剪：

### 输入

主 agent 在 prompt 中已声明：
- 团队仓相对路径：`_prd-tools/distill/{slug}/_ingest/prd.md`
- 团队仓相对路径：`_prd-tools/distill/{slug}/context/requirement-ir.yaml`
- 角色 hint：`role={producer|consumer|middleware}`

subagent 启动后**先 cd 到 `repos/{repo}/`**（团队仓内的 submodule 工作树），CWD 即该成员仓源码根目录。`references/{repo}/` 在团队仓根的相对路径仍可读。

### 跳过的步骤

| 步骤 | 处理 |
|------|------|
| Step 1 PRD Ingestion | **跳过**（主 agent 已生成 `_ingest/prd.md`） |
| Step 2 Evidence | **跳过**（主 agent 已生成 evidence） |
| Step 3 Requirement IR | **跳过**（主 agent 已生成 `requirement-ir.yaml`） |
| Step 4 Code Search & Layer Impact | **正常执行**，可用 rg/glob |
| Step 5 Contract Delta | **正常执行**，但 `producer/consumers` 仅基于本仓 03-contracts |
| Step 6 Report Confirmation | **正常执行** |
| Step 7 Report 生成 | **正常执行** |
| Step 8 Plan | **跳过**（plan 由主 agent 统一生成） |
| Step 9 Readiness | **跳过** |
| Step 10 Reference Backflow | **跳过**（建议留给主 agent 聚合后统一处理） |
| Step 11 Quality Gate | **跳过**（团队 quality-gate 在主 agent 末段统一跑） |

### 输出路径改写

所有产物根目录改为：`_prd-tools/distill/{slug}/per-repo/{repo}/`（路径相对**团队仓根**，subagent 需写绝对路径或返回团队仓根）。

具体落点：
- `_prd-tools/distill/{slug}/per-repo/{repo}/report.md`
- `_prd-tools/distill/{slug}/per-repo/{repo}/context/{layer-impact, contract-delta, graph-context, report-confirmation}.yaml`
- `_prd-tools/distill/{slug}/per-repo/{repo}/evidence/`

### 失败处理

任何 step 失败时，**不重试**，写入 `_prd-tools/distill/{slug}/per-repo/{repo}/_failure.json`：

```json
{ "repo": "<repo>", "failed_at_step": "Step 4", "error": "<message>", "partial_outputs": ["report.md"] }
```

并在 subagent 返回值中标记 `status: failed`，由主 agent 把该仓标记为 unavailable。

### 边界（必须遵守）

- ❌ 不读 `references/{other-repo}/`，不读 `repos/{other-repo}/`
- ❌ 不生成 plan、readiness、quality-gate 产物
- ❌ 不做跨仓推理（producer 是否被某仓 consumer、handoff 闭环、owner 缺位 — 全留给主 agent）
- ✅ contract-delta 中 `producer` 字段只填本仓视角；跨仓信息留空，由主 agent 补
````

- [ ] **Step 3: 验证追加成功**

```bash
grep -n "single-repo subagent 模式" plugins/prd-distill/skills/prd-distill/workflow.md
```

预期至少 2 行（一行为标题，一行为正文出现）。

```bash
python3 -c "
content = open('plugins/prd-distill/skills/prd-distill/workflow.md').read()
assert '## 附：single-repo subagent 模式' in content
assert 'per-repo/{repo}/' in content
assert '不重试' in content
print('OK')
"
```

预期 `OK`。

- [ ] **Step 4: Commit**

```bash
git add plugins/prd-distill/skills/prd-distill/workflow.md
git commit -m "feat(prd-distill): add single-repo subagent mode for team-distill fan-out"
```

---

## Task 4: team-distill SKILL.md 差异表更新

**Files:**
- Modify: `plugins/prd-distill/skills/team-distill/SKILL.md:14-23` 的差异表

- [ ] **Step 1: 阅读当前差异表**

```bash
sed -n '12,24p' plugins/prd-distill/skills/team-distill/SKILL.md
```

确认表格 8 行：源码扫描、Step 4.1、Step 4.2、涉及仓库识别、Contract Delta、Plan、Report。

- [ ] **Step 2: 整张表替换为升级后版本**

定位 SKILL.md 中以下整段：

```markdown
| 方面 | 单仓 `/prd-distill` | 团队 `/team-distill` |
|------|---------------------|---------------------|
| 源码扫描 | rg/glob + reference | **禁止 rg/glob**，只读 `references/{repo}/` 下的 01-05 YAML |
| Step 4.1 Query Plan | index 存在则执行 | 从 `references/{repo}/index/` 加载多仓 index（`context-pack.py --team-references`） |
| Step 4.2 Graph Context | 3 阶段扫描 | **只读 reference**：从各仓 01-05 YAML 构建理解，禁止 rg/glob |
| 涉及仓库识别 | 不适用 | 自动匹配 PRD 需求 → 各仓 reference，识别涉及仓库及角色 |
| Contract Delta | 单仓视角 | 跨仓视角：从各仓 03-contracts.yaml 理解 producer/consumer 边界 |
| Plan | 1 份 plan.md | 1 份 team-plan.md + N 份 plans/plan-{repo}.md |
| Report | 1 份 report.md | 1 份 report.md（按仓库分组展示影响和任务） |

成员仓列表来自 `project-profile.yaml` 的 `team_repos[]`。涉及的仓库和角色从各仓 03-contracts.yaml 自动推断。
```

替换为：

```markdown
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
```

- [ ] **Step 3: 同步更新「输出结构」节**

定位 SKILL.md 的 "输出结构" 章节：

```markdown
```text
_prd-tools/distill/<slug>/
├── _ingest/                       # 同单仓
├── report.md                      # 团队级报告（§9 分 5 个子节）
├── team-plan.md                   # 团队级开发计划总览
├── plans/                         # Sub-Plans（动态命名）
│   ├── plan-{repo1}.md            # 成员仓 sub-plan
│   └── plan-{repo2}.md
└── context/
    ├── layer-impact.yaml          # 4 层完整填充
    ├── contract-delta.yaml        # 跨仓 consumers[]
    └── ...                        # 其余同单仓
```
```

替换为：

```markdown
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
```

- [ ] **Step 4: 验证 grep**

```bash
grep -n "fan-out/fan-in\|cross-align\|per-repo/\|Step 3.5\|Step 7.5" plugins/prd-distill/skills/team-distill/SKILL.md
```

预期至少 5 行命中（Step 3.5 / Step 7.5 / per-repo/ / cross-align / fan-out/fan-in）。

- [ ] **Step 5: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/SKILL.md
git commit -m "docs(team-distill): update SKILL.md to reflect fan-out/fan-in architecture"
```

---

## Task 5: team-distill workflow.md 重写（核心改动）

**Files:**
- Modify: `plugins/prd-distill/skills/team-distill/workflow.md`（整体重写主体）

为避免一次大改动难审，分 5 个 sub-task 完成。

### Task 5a: 重写文件 header 与「Step 1-3 / Step 3.5」章节

- [ ] **Step 1: 备份当前 workflow.md（仅本 task 链工作目录用，不 commit）**

```bash
cp plugins/prd-distill/skills/team-distill/workflow.md /tmp/team-distill-workflow-backup.md
```

- [ ] **Step 2: 替换文件头到「Step 4」之前的内容**

定位 `plugins/prd-distill/skills/team-distill/workflow.md` 开头到 `## Step 4：Code Search & Layer Impact（团队模式）` 之前，全部替换为：

```markdown
# team-distill 工作流

> **架构**：主 agent 编排 + subagent 并行蒸馏 + 主 agent 聚合（fan-out/fan-in）。spec 见 [docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md](../../../../docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md)。
>
> Step 1-3（PRD Ingestion / Evidence / Requirement IR）由**主 agent**执行，与单仓模式相同；详见 [skills/prd-distill/workflow.md](../../prd-distill/skills/prd-distill/workflow.md)。
> Step 4-7 由 **subagent fan-out** 执行（每仓一个 subagent，使用 prd-distill 的 "single-repo subagent 模式"）。
> Step 7.5 / 7.6 / 8 / 9-11 由**主 agent**聚合。

## 目标

面向多仓库团队的 PRD 蒸馏：

1. 主 agent 在团队仓根解析 PRD、生成 requirement-ir、识别涉及仓
2. 对每个涉及仓 fan-out 一个 subagent，subagent 在 `repos/{repo}/` 源码上跑 Step 4-7
3. 主 agent 聚合各仓产物，跨仓对齐契约，生成团队 report + team-plan + sub-plans

前置：`project-profile.yaml` 含 `layer: "team-common"` 且 `team_repos[]` 每条配置了 `source_path` 与 `submodule: true`，且团队仓根有 `references/{repo}/` 与 `repos/{repo}/`（submodule 已 init）。

---

## Step 1-3：主 agent 一次性完成

PRD Ingestion → Evidence → Requirement IR，流程同 [prd-distill/workflow.md](../../prd-distill/skills/prd-distill/workflow.md)。

额外消费（同旧版）：
- 各仓 `references/{repo}/05-domain.yaml`：术语，用于 requirement-ir 术语对齐
- 各仓 `references/{repo}/02-coding-rules.yaml`：fatal 规则

产物落到团队仓根的 `_prd-tools/distill/{slug}/`：
- `_ingest/prd.md`、`_ingest/document.md`、`_ingest/document-structure.json`
- `context/requirement-ir.yaml`
- `evidence/EV-*.yaml`

## Step 3.5：涉及仓识别（新增，主 agent）

输入：`requirement-ir.yaml` + 各仓 `references/{repo}/{01-codebase, 03-contracts, 04-routing-playbooks}.yaml`

算法（启发式，确定性）：

1. 对每个 REQ 提取关键词集合：实体名、动作名、路由片段
2. 与各仓 `01-codebase.yaml` 的 modules/entities 名称匹配 → 命中标记该仓相关
3. 与各仓 `04-routing-playbooks.yaml` 的 routes/handoffs 匹配 → 命中标记该仓相关
4. 角色推断：依据各仓 `03-contracts.yaml` 的 `producer`/`consumers[]` 字段
5. 同一仓多 REQ 命中合并

产物：`context/involved-repos.yaml`

```yaml
schema_version: "1"
involved_repos:
  - repo: dive-bff
    role: middleware                    # producer | consumer | middleware
    matched_via: ["module:order", "route:/api/v1/order"]
    confidence: high                    # high | medium
    matched_reqs: ["REQ-001", "REQ-003"]
not_involved_repos:
  - repo: dive-be-legacy
    reason: "无关键词命中"
```

漏判兜底：Step 7.5 cross-align 阶段如果发现 `consumer_orphan`，在 §9.5 提示"疑似漏 fan-out 仓"。

```

- [ ] **Step 3: 验证文件头部正确**

```bash
python3 -c "
content = open('plugins/prd-distill/skills/team-distill/workflow.md').read()
assert '## Step 1-3：主 agent 一次性完成' in content
assert '## Step 3.5：涉及仓识别' in content
assert 'involved_repos' in content
assert 'fan-out/fan-in' in content
print('OK')
"
```

预期 `OK`。

- [ ] **Step 4: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/workflow.md
git commit -m "refactor(team-distill): rewrite header + add Step 3.5 (involved repo detection)"
```

### Task 5b: 重写「Step 4-7 fan-out」章节

- [ ] **Step 1: 替换 Step 4-7 整段**

定位文件中 `## Step 4：Code Search & Layer Impact（团队模式）` 到 `## Step 6-7：Report（团队模式）` 章节末尾（不含 `## Step 8`）的整段，全部替换为：

```markdown
## Step 4-7：Fan-out 到 subagent

主 agent 不再亲自跑 Step 4-7。对 `Step 3.5` 输出的每个 `involved_repos[i]`，并行 dispatch 一个 subagent。

### 4-7.A 启动 subagent

主 agent 用 `Agent` 工具调用，subagent_type=`general-purpose`，prompt 模板如下（**字面量必须保留**，特别是 `single-repo subagent 模式` 这个标志短语）：

```
你正在 /team-distill 的 fan-out 阶段，负责仓库：{repo}。

工作目录（CWD 启动后切换）：repos/{repo}/   ← 团队仓内的 submodule，真实源码
参考资料：<团队仓根>/references/{repo}/      ← 该仓 reference

输入产物路径（相对团队仓根，绝对路径请用 <团队仓根> 开头）：
- _prd-tools/distill/{slug}/_ingest/prd.md
- _prd-tools/distill/{slug}/context/requirement-ir.yaml
- 你被识别为：role={producer|consumer|middleware}（hint，可推翻）

任务：调用 superpowers:prd-distill skill，按 "single-repo subagent 模式" 跑 Step 4-7
（详见 prd-distill/workflow.md 末尾"附：single-repo subagent 模式"）。

**禁止跑 Step 1-3 与 Step 8-11**。

输出路径（相对团队仓根）：
_prd-tools/distill/{slug}/per-repo/{repo}/

成功时返回 JSON：
{ "repo": "{repo}", "status": "ok", "output_dir": "_prd-tools/distill/{slug}/per-repo/{repo}/", "summary": "..." }

失败时返回 JSON 并写 _failure.json：
{ "repo": "{repo}", "status": "failed", "errors": [...] }
```

并行规则：
- 所有 subagent 同时启动（不分批）
- 等所有 subagent 返回后（barrier），再进入 Step 7.5

### 4-7.B 失败处理

| 情况 | 主 agent 动作 |
|------|--------------|
| `repos/{repo}/` 不存在或 submodule 未 init | 跳过 fan-out，加入 `unavailable_repos[]`，原因 `submodule_uninitialized` |
| `repos/{repo}` HEAD ≠ `references/{repo}` 的 `git_head` | 仍然 fan-out，但在 §9.{repo} 顶部加 `head_drift: source=<sha1> reference=<sha2>` |
| subagent 返回 `status: failed` 或超时 | 不重试，加入 `unavailable_repos[]`，原因 `subagent_failed` |
| subagent 返回 `status: ok` 但缺关键产物（report.md / contract-delta.yaml） | 视为 partial，加入 `partial_repos[]`，confidence 降级 |

### 4-7.C 主 agent 不做的事

- 不读 `repos/{repo}/` 任何源码（这是 subagent 的事）
- 不为 subagent 复审 layer-impact / contract-delta（subagent 自己出，主 agent 只做跨仓聚合）
- 不重试任何 subagent

### 4-7.D 输出验收

每个 `involved_repos[i]` 完成后，主 agent 校验 `per-repo/{repo}/` 至少包含：
- `report.md`（非空）
- `context/layer-impact.yaml`（非空）
- `context/contract-delta.yaml`（可空，但文件需存在；空时记录 `no_contract_changes: true`）

缺少则降级到 `partial_repos[]`。
```

- [ ] **Step 2: 验证**

```bash
python3 -c "
content = open('plugins/prd-distill/skills/team-distill/workflow.md').read()
assert '## Step 4-7：Fan-out 到 subagent' in content
assert 'single-repo subagent 模式' in content
assert 'unavailable_repos' in content
assert 'head_drift' in content
print('OK')
"
```

预期 `OK`。

- [ ] **Step 3: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/workflow.md
git commit -m "refactor(team-distill): replace Step 4-7 with subagent fan-out orchestration"
```

### Task 5c: 加「Step 7.5 跨仓对齐」章节

- [ ] **Step 1: 在 4-7 章节之后、Step 8 章节之前插入 Step 7.5**

定位文件中 `## Step 8：Plan（团队模式）` 之前，插入：

```markdown
## Step 7.5：跨仓对齐（新增，主 agent）

输入：所有 `per-repo/{repo}/context/contract-delta.yaml`（每个 subagent 已产出本仓视角）

算法（确定性，不依赖 LLM 推理；按规则执行）：

1. **Endpoint 索引**
   遍历每个 `per-repo/{repo}/context/contract-delta.yaml`，对每条 `deltas[i]`：
   - 抽 key：`(deltas[i].name, deltas[i].contract_surface)`（例如 `("POST /api/v1/order", "endpoint")`）
   - 记录：`{repo, role: producer if deltas[i].producer == repo's layer else consumer}`

2. **闭环检查** — 按 key 分组：
   | 出现情况 | 标记 |
   |---------|------|
   | 同 key 同时有 producer 和至少 1 个 consumer | `aligned` |
   | 只有 producer，无 consumer | `producer_orphan`（可能正常，记录但不报警） |
   | 只有 consumer，无 producer | `consumer_orphan`（**风险**） |
   | 多个 producer | `producer_conflict`（**风险**） |

3. **字段一致性**
   同 key producer 与各 consumer 的 `request_fields[]` / `response_fields[]` 字段名集合做差集，写入 `field_drift[]`：
   ```yaml
   field_drift:
     - key: "POST /api/v1/order"
       only_in_producer: ["x_signature"]
       only_in_consumer: ["legacy_id"]
   ```

4. **Owner 缺位**
   对 `aligned` 的 endpoint，查 producer 仓 `references/{repo}/03-contracts.yaml` 该条目的 `owner` — 为空标记 `owner_missing`。

5. **Handoff 合并**
   读各仓 `references/{repo}/04-routing-playbooks.yaml` 的 `cross_repo_handoffs[]`，按 `(from_repo, to_repo, surface)` 三元组去重合并。

### 产物：`context/cross-align.yaml`

详细 schema 见 [prd-distill/references/output-contracts.md](../../prd-distill/skills/prd-distill/references/output-contracts.md) 的 `context/cross-align.yaml` 章节。简要结构：

```yaml
schema_version: "1"
endpoints:
  - key: "POST /api/v1/order"
    status: aligned | consumer_orphan | producer_conflict | producer_orphan
    producer: { repo: dive-bff, owner: "team-bff" }
    consumers: [{ repo: dive-fe }]
    field_drift: { only_in_producer: [], only_in_consumer: [] }
    risks: ["owner_missing"]
handoffs:
  - from_repo: dive-fe
    to_repo: dive-bff
    surface: "POST /api/v1/order"
unavailable_repos: ["dive-be-legacy"]
suspected_missing_fanout: []   # consumer_orphan 推断出的疑似漏 fan-out 仓
```

### 漏判兜底

如有 `consumer_orphan` 且对应 endpoint 在某 `not_involved_repos[]` 仓的 `references/{repo}/03-contracts.yaml` 中声明为 producer → 加入 `suspected_missing_fanout[]`。最终 §9.5 提示用户重跑该仓。
```

- [ ] **Step 2: 验证**

```bash
python3 -c "
content = open('plugins/prd-distill/skills/team-distill/workflow.md').read()
assert '## Step 7.5：跨仓对齐' in content
assert 'consumer_orphan' in content
assert 'producer_conflict' in content
assert 'suspected_missing_fanout' in content
print('OK')
"
```

预期 `OK`。

- [ ] **Step 3: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/workflow.md
git commit -m "feat(team-distill): add Step 7.5 cross-repo contract alignment"
```

### Task 5d: 加「Step 7.6 Report 聚合」章节

- [ ] **Step 1: 在 Step 7.5 之后、Step 8 之前插入 Step 7.6**

```markdown
## Step 7.6：Report 聚合（主 agent）

`report.md` 顶层结构（11 节标准模板，§9 子节按团队仓扩展）：

| 章节 | 来源 |
|------|------|
| §1-§8 通用章节 | 主 agent 基于 requirement-ir + 跨仓视角生成 |
| §9.1 Frontend | 摘要 `per-repo/{role=frontend 的仓}/report.md` 的 §3 + §6 |
| §9.2 BFF | 摘要 `per-repo/{role=bff 的仓}/report.md` |
| §9.3 Backend | 摘要 `per-repo/{role=backend 的仓}/report.md` |
| §9.4 External | 主 agent 从各仓 layer-impact 的 external 层合并 |
| §9.5 跨层对齐风险 | 直接由 `cross-align.yaml` 渲染 |
| §9.{unavail-repo} | 标 `unavailable: <reason>`，confidence=low |

### 摘要规则

每个 §9.{repo} 子节写：

1. **代码坐标**：列前 5 个 IMP 的 anchor（指向 `repos/{repo}/...` 的真实路径）
2. **关键契约 delta**：从 per-repo contract-delta.yaml 抽取的 `change_type != NO_CHANGE` 条目
3. **风险摘要**：复制 per-repo report.md 的 §6 风险节顶部 3 条
4. **锚点链接**：`详见 [per-repo/{repo}/report.md](per-repo/{repo}/report.md)`

**禁止全文复制 per-repo report.md** — 全文留在 `per-repo/{repo}/report.md`。

### Layer Impact 聚合

主 agent 把各仓 `per-repo/{repo}/context/layer-impact.yaml` 的 4 层条目合并，写入 `context/layer-impact.yaml`，每个 IMP 加 `repo:` 字段标识来源仓。

### Contract Delta 聚合

主 agent 把各仓 `per-repo/{repo}/context/contract-delta.yaml` 的 deltas[] 合并到顶层 `context/contract-delta.yaml`，每条加 `repo:` 字段；`consumers[]` 按 cross-align 结果跨仓填充。
```

- [ ] **Step 2: 验证**

```bash
python3 -c "
content = open('plugins/prd-distill/skills/team-distill/workflow.md').read()
assert '## Step 7.6：Report 聚合' in content
assert '禁止全文复制' in content
assert '§9.{unavail-repo}' in content
print('OK')
"
```

预期 `OK`。

- [ ] **Step 3: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/workflow.md
git commit -m "feat(team-distill): add Step 7.6 report aggregation"
```

### Task 5e: 重写「Step 8 Plan」与「Step 9-11」尾部

- [ ] **Step 1: 替换 Step 8 章节**

定位文件中 `## Step 8：Plan（团队模式）` 整段（到下一节为止），替换为：

```markdown
## Step 8：Plan 生成（主 agent）

主 agent **统一生成**所有 plan，subagent 不出 plan。

### team-plan.md 结构（沿用 7 节，内容来源升级）

1. **范围与假设**：目标、跨仓依赖、`involved_repos` 角色表（来自 Step 3.5）
2. **涉及仓库总览**：每仓代码坐标（从 per-repo layer-impact 抽 anchor）+ 跨仓调用链（从 cross-align.handoffs）
3. **跨仓时序**：依赖图由 `cross-align.endpoints` 的 producer→consumer 关系生成；Phase 1-N 按"被依赖在前"拓扑排序
4. **Sub-Plan 索引表**：列每个 plan 文件路径 + IMP 数 + 是否 unavailable
5. **契约对齐（跨仓）**：从 `cross-align.yaml` 抽 `aligned/orphan/conflict/drift` 摘要
6. **风险与回滚**：跨仓联调风险（cross-align 中 risks 字段聚合）+ 回滚策略
7. **工作量总览**：按仓汇总（从 per-repo report 工作量节加总）

### plans/plan-{repo}.md

主 agent 基于 `per-repo/{repo}/report.md` 生成，沿用单仓 11-section plan 模板。**scope 限定到该仓**，不混入其他仓信息。

unavailable 仓也要产出占位 plan：

```markdown
# plan-{repo}.md (UNAVAILABLE)

> 该仓在本次蒸馏中标记为 unavailable，原因：{reason}
> 待用户处理后，重跑 `/team-distill` 生成完整 plan。

**Status:** blocked
**Next action:** {next_action_text}
```

文件名从 `team_repos[].repo` 动态生成，禁止硬编码。

## Step 9：Readiness Score（主 agent）

同单仓模式，但分子分母按 involved_repos 计算（unavailable 仓不计入分母）。

## Step 10：Reference Backflow（主 agent）

同单仓模式，但每条建议加 `target_repo` 字段。team_reference_candidate 仍标记，不自动写入。

## Step 11：Quality Gate（主 agent）

```bash
python3 scripts/quality-gate.py distill \
  --distill-dir _prd-tools/distill/<slug> \
  --repo-root .
```

检查项见 [Task 6 quality-gate 扩展](#)。unavailable 仓不阻断交付，但 gate 输出 `severity: warning` 计入摘要。
```

- [ ] **Step 2: 验证整个 workflow.md 结构连贯**

```bash
python3 << 'EOF'
import re
content = open('plugins/prd-distill/skills/team-distill/workflow.md').read()

# 步骤编号必须连续：1-3 / 3.5 / 4-7 / 7.5 / 7.6 / 8 / 9 / 10 / 11
steps = re.findall(r'^## Step (\S+?)[：:]', content, re.M)
print('Step headers:', steps)
expected = ['1-3', '3.5', '4-7', '7.5', '7.6', '8', '9', '10', '11']
assert steps == expected, f'步骤标题不符合预期: {steps} != {expected}'

# CLAUDE.md 编号规范：禁止 2.5/3.5/8.1 但 3.5/7.5/7.6 在本次设计中允许
# 因为它们是「插入步骤」标记新增的逻辑，与 4.1/4.2/4.3 同为子级标记 — 但本设计已经文档化为
# 顶级 Step 命名。检查没有 0 或负数。
for s in steps:
    assert not s.startswith('0'), f'禁止 Step 0: {s}'

# 关键短语
for kw in ['fan-out/fan-in', 'subagent', 'cross-align', 'per-repo/']:
    assert kw in content, f'缺关键短语: {kw}'

print('OK')
EOF
```

预期 `OK`。

> **注**：CLAUDE.md "步骤编号规范" 禁止 2.5/3.5/8.1 这类小数。本设计的 Step 3.5 / 7.5 / 7.6 是**显式插入**的新流程节点，与单仓 prd-distill 的现有步骤编号衔接。建议在本次实施时，**同步更新 CLAUDE.md 的步骤编号规范**，给"团队模式插入步骤"开口子（或在 plan 完成后主 agent 单开 PR 调整规范）。本 plan 不在内文擅自改 CLAUDE.md。

- [ ] **Step 3: Commit**

```bash
git add plugins/prd-distill/skills/team-distill/workflow.md
git commit -m "refactor(team-distill): rewrite Step 8/9/10/11 for fan-out/fan-in mode"
```

---

## Task 6: 在 output-contracts.md 加 cross-align.yaml schema 章节

**Files:**
- Modify: `plugins/prd-distill/skills/prd-distill/references/output-contracts.md`

- [ ] **Step 1: 定位 contract-delta.yaml 章节末尾位置**

```bash
grep -n "## context/contract-delta.yaml\|## context/reference-update-suggestions.yaml" plugins/prd-distill/skills/prd-distill/references/output-contracts.md
```

记录两个章节的行号（contract-delta 和它的下一节）。

- [ ] **Step 2: 在 contract-delta.yaml 章节之后、reference-update-suggestions.yaml 之前插入 cross-align.yaml schema**

````markdown

## context/cross-align.yaml（团队模式专用）

`/team-distill` Step 7.5 由主 agent 产出。subagent 不产此文件。

```yaml
schema_version: "1"
tool_version: "<tool-version>"
meta:
  primary_source: "per-repo/{repo}/context/contract-delta.yaml"
  produced_by: "team-distill main agent (Step 7.5)"
endpoints:
  - key: "POST /api/v1/order"            # name + contract_surface 组合
    contract_surface: "endpoint"          # endpoint | schema | event | payload | db_table | external_api
    status: "aligned | consumer_orphan | producer_conflict | producer_orphan"
    producer:
      repo: "dive-bff"
      owner: "team-bff"
    consumers:
      - repo: "dive-fe"
    field_drift:
      only_in_producer: []
      only_in_consumer: []
    risks:                                # owner_missing | field_drift | producer_conflict | ...
      - "owner_missing"
handoffs:                                  # 各仓 04-routing-playbooks.cross_repo_handoffs 合并去重
  - from_repo: "dive-fe"
    to_repo: "dive-bff"
    surface: "POST /api/v1/order"
unavailable_repos:
  - repo: "dive-be-legacy"
    reason: "submodule_uninitialized | subagent_failed"
suspected_missing_fanout: []              # consumer_orphan 推断出的疑似漏 fan-out 仓
summary:
  total_endpoints: 0
  aligned: 0
  consumer_orphan: 0
  producer_conflict: 0
  producer_orphan: 0
```

生成规则：

- 仅在 `/team-distill` 团队模式产出，单仓 `/prd-distill` 不产
- `endpoints[].key` = `<name> <contract_surface>` 拼接，确保跨仓 endpoint 可对齐
- `risks[]` 来源：`owner_missing` 由产 producer 仓的 `references/{repo}/03-contracts.yaml` owner 字段判定；`field_drift` 由 producer/consumer 字段 diff 判定
- `suspected_missing_fanout[]` 仅在以下条件成立时才填入：`consumer_orphan` 的 endpoint 在某个 `Step 3.5` 标为 not_involved 的仓的 `references/{repo}/03-contracts.yaml` 中声明为 producer
````

- [ ] **Step 3: 验证**

```bash
python3 -c "
content = open('plugins/prd-distill/skills/prd-distill/references/output-contracts.md').read()
assert '## context/cross-align.yaml' in content
assert 'consumer_orphan' in content
assert 'suspected_missing_fanout' in content
import yaml
# 提取章节中的 yaml 代码块解析
import re
yaml_blocks = re.findall(r'```yaml\n(.*?)```', content, re.S)
target = [b for b in yaml_blocks if 'producer_orphan' in b and 'unavailable_repos' in b]
assert len(target) == 1, f'cross-align yaml block not unique: {len(target)}'
yaml.safe_load(target[0])  # schema 必须是合法 yaml
print('OK')
"
```

预期 `OK`。

- [ ] **Step 4: Commit**

```bash
git add plugins/prd-distill/skills/prd-distill/references/output-contracts.md
git commit -m "docs(prd-distill): add cross-align.yaml schema for team-distill Step 7.5"
```

---

## Task 7: quality-gate.py 扩展 4 项检查

**Files:**
- Modify: `scripts/quality-gate.py`（追加函数 + `run_distill_quality` 集成）

### Task 7a: 加 `_dq_per_repo_completeness` 检查

- [ ] **Step 1: 阅读现有 `_dq_team_sub_plans` 函数定位插入点**

```bash
sed -n '214,225p' scripts/quality-gate.py
```

确认末尾在第 220 行附近。新函数将插入到 `_dq_team_sub_plans` 之后、`_dq_prd_coverage_simple` 之前。

- [ ] **Step 2: 写测试断言（先验证当前未实现）**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('qg', 'scripts/quality-gate.py')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
assert not hasattr(mod, '_dq_per_repo_completeness'), '函数已存在，预期未实现'
print('未实现，可以写')
"
```

预期 `未实现，可以写`。

- [ ] **Step 3: 在 `_dq_team_sub_plans` 函数之后追加新函数**

定位 `scripts/quality-gate.py` 中：

```python
def _dq_team_sub_plans(base, member_repos):
    found, missing = [], []
    for repo in member_repos:
        pf = base / 'plans' / f'plan-{repo}.md'
        (found if file_exists_nonempty(pf) else missing).append(repo)
    return {'status': 'pass' if not missing else 'warning',
            'sub_plans_found': found, 'sub_plans_missing': missing}
```

在它之后追加：

```python
def _dq_per_repo_completeness(base, involved_repos):
    """检查 per-repo/{repo}/ 完整性。

    每个 involved_repo 必须有 report.md（非空）和 context/{layer-impact, contract-delta}.yaml 文件存在。
    若 _failure.json 存在则视为 unavailable，不计入缺失。
    """
    per_repo_dir = base / 'per-repo'
    if not per_repo_dir.is_dir():
        return {'status': 'fail', 'reason': 'per-repo/ 目录不存在', 'expected_repos': involved_repos}
    incomplete, unavailable, complete = [], [], []
    for repo in involved_repos:
        d = per_repo_dir / repo
        if (d / '_failure.json').is_file():
            unavailable.append(repo)
            continue
        if not d.is_dir():
            incomplete.append({'repo': repo, 'reason': 'missing_dir'})
            continue
        report = d / 'report.md'
        li = d / 'context' / 'layer-impact.yaml'
        cd = d / 'context' / 'contract-delta.yaml'
        if not file_exists_nonempty(report):
            incomplete.append({'repo': repo, 'reason': 'missing_report'})
        elif not li.is_file():
            incomplete.append({'repo': repo, 'reason': 'missing_layer_impact'})
        elif not cd.is_file():
            incomplete.append({'repo': repo, 'reason': 'missing_contract_delta'})
        else:
            complete.append(repo)
    status = 'fail' if incomplete else ('warning' if unavailable else 'pass')
    return {
        'status': status,
        'complete': complete,
        'incomplete': incomplete,
        'unavailable': unavailable,
    }
```

- [ ] **Step 4: 验证函数定义可被 import**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('qg', 'scripts/quality-gate.py')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
assert hasattr(mod, '_dq_per_repo_completeness')
import inspect
sig = inspect.signature(mod._dq_per_repo_completeness)
assert list(sig.parameters) == ['base', 'involved_repos']
print('OK')
"
```

预期 `OK`。

- [ ] **Step 5: 写小型 fixture 验证返回值**

```bash
python3 << 'EOF'
import importlib.util, tempfile, json
from pathlib import Path

spec = importlib.util.spec_from_file_location('qg', 'scripts/quality-gate.py')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

with tempfile.TemporaryDirectory() as tmp:
    base = Path(tmp)
    # repo-A 完整
    (base / 'per-repo' / 'A' / 'context').mkdir(parents=True)
    (base / 'per-repo' / 'A' / 'report.md').write_text('content')
    (base / 'per-repo' / 'A' / 'context' / 'layer-impact.yaml').write_text('x: 1')
    (base / 'per-repo' / 'A' / 'context' / 'contract-delta.yaml').write_text('x: 1')
    # repo-B 失败
    (base / 'per-repo' / 'B').mkdir(parents=True)
    (base / 'per-repo' / 'B' / '_failure.json').write_text('{}')
    # repo-C 缺 report
    (base / 'per-repo' / 'C' / 'context').mkdir(parents=True)
    (base / 'per-repo' / 'C' / 'context' / 'layer-impact.yaml').write_text('x: 1')

    r = mod._dq_per_repo_completeness(base, ['A', 'B', 'C'])
    assert r['status'] == 'fail', r
    assert r['complete'] == ['A'], r
    assert r['unavailable'] == ['B'], r
    assert any(x['repo'] == 'C' and x['reason'] == 'missing_report' for x in r['incomplete']), r
    print('OK:', r)
EOF
```

预期最后一行 `OK: {...}`。

- [ ] **Step 6: Commit**

```bash
git add scripts/quality-gate.py
git commit -m "feat(quality-gate): add _dq_per_repo_completeness check"
```

### Task 7b: 加 `_dq_cross_align` 与 `_dq_team_section_9` 检查

- [ ] **Step 1: 在 `_dq_per_repo_completeness` 之后追加两个函数**

```python
def _dq_cross_align(base, has_contract_changes):
    """检查 cross-align.yaml 存在性与基本结构。

    - 文件不存在：若 has_contract_changes=True 则 warning（PRD 有契约改动但无对齐结论），否则 pass
    - 文件存在但无 endpoints[]：warning
    - 文件存在且有 endpoints[]：pass，附统计
    """
    p = base / 'context' / 'cross-align.yaml'
    if not p.is_file():
        if has_contract_changes:
            return {'status': 'warning', 'reason': 'cross-align.yaml 缺失但 PRD 涉及契约改动'}
        return {'status': 'pass', 'reason': 'PRD 无契约改动，cross-align 可省略'}
    try:
        data = yaml.safe_load(p.read_text(encoding='utf-8')) or {}
    except Exception as e:
        return {'status': 'fail', 'reason': f'yaml 解析失败: {e}'}
    endpoints = data.get('endpoints') or []
    if not endpoints:
        return {'status': 'warning', 'reason': 'cross-align.yaml 存在但 endpoints 为空'}
    by_status = {}
    for ep in endpoints:
        s = ep.get('status', 'unknown')
        by_status[s] = by_status.get(s, 0) + 1
    suspected = data.get('suspected_missing_fanout') or []
    status = 'warning' if (by_status.get('consumer_orphan', 0) > 0 or suspected) else 'pass'
    return {
        'status': status,
        'total': len(endpoints),
        'by_status': by_status,
        'suspected_missing_fanout': suspected,
    }


def _dq_team_section_9(base, involved_repos):
    """检查 report.md §9 子节齐全。

    必须含 §9.1 Frontend / §9.2 BFF / §9.3 Backend / §9.4 External / §9.5 跨层对齐风险，
    以及每个 involved_repo 一个 §9.{repo} 子节（unavailable 仓也要有占位）。
    """
    p = base / 'report.md'
    if not file_exists_nonempty(p):
        return {'status': 'fail', 'reason': 'report.md 不存在或为空'}
    text = p.read_text(encoding='utf-8')
    required_titles = ['§9.1 Frontend', '§9.2 BFF', '§9.3 Backend', '§9.4 External', '§9.5']
    missing = [t for t in required_titles if t not in text]
    repo_missing = [r for r in involved_repos if f'§9.{r}' not in text and f'### {r}' not in text]
    status = 'fail' if missing else ('warning' if repo_missing else 'pass')
    return {
        'status': status,
        'missing_required': missing,
        'missing_repo_subsections': repo_missing,
    }
```

- [ ] **Step 2: 验证函数解析与 fixture 测试**

```bash
python3 << 'EOF'
import importlib.util, tempfile, yaml as y
from pathlib import Path

spec = importlib.util.spec_from_file_location('qg', 'scripts/quality-gate.py')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

with tempfile.TemporaryDirectory() as tmp:
    base = Path(tmp)
    # cross-align: 不存在 + has_contract_changes=True → warning
    r1 = mod._dq_cross_align(base, has_contract_changes=True)
    assert r1['status'] == 'warning', r1
    # cross-align 存在且 healthy
    (base / 'context').mkdir()
    (base / 'context' / 'cross-align.yaml').write_text(y.safe_dump({
        'endpoints': [{'status': 'aligned'}],
        'suspected_missing_fanout': [],
    }))
    r2 = mod._dq_cross_align(base, has_contract_changes=True)
    assert r2['status'] == 'pass', r2

    # report.md 缺 §9.5
    (base / 'report.md').write_text('# Report\n## §9\n### §9.1 Frontend\n### §9.2 BFF\n### §9.3 Backend\n### §9.4 External\n')
    r3 = mod._dq_team_section_9(base, ['dive-bff'])
    assert r3['status'] == 'fail', r3
    assert '§9.5' in r3['missing_required'], r3
    print('OK')
EOF
```

预期 `OK`。

- [ ] **Step 3: Commit**

```bash
git add scripts/quality-gate.py
git commit -m "feat(quality-gate): add _dq_cross_align and _dq_team_section_9 checks"
```

### Task 7c: 集成到 `run_distill_quality`

- [ ] **Step 1: 阅读 `run_distill_quality` 当前实现**

```bash
sed -n '238,256p' scripts/quality-gate.py
```

确认结构。我们要在 team-mode 分支加 3 个新检查。

- [ ] **Step 2: 修改 `run_distill_quality`**

定位：

```python
def run_distill_quality(base, repo_root):
    is_team, member_repos = detect_team_mode(repo_root)
    results = {
        'required_files': _dq_required_files(base, is_team=is_team),
        'requirement_ir': _dq_requirement_ir(base),
        'layer_impact': _dq_layer_impact(base),
        'index_bridge': _dq_index_bridge(base, repo_root),
        'final_quality_gate': _dq_final_quality_gate(base),
        'report_quality': _dq_report_quality(base),
        'report_confirmation': _dq_report_confirmation(base),
        'plan_missing_confirmation': _dq_plan_missing_confirmation(base, is_team=is_team),
    }
    if is_team:
        results['team_sub_plans'] = _dq_team_sub_plans(base, member_repos)
    results['prd_coverage'] = _dq_prd_coverage_simple(base)
    return results
```

替换为：

```python
def run_distill_quality(base, repo_root):
    is_team, member_repos = detect_team_mode(repo_root)
    results = {
        'required_files': _dq_required_files(base, is_team=is_team),
        'requirement_ir': _dq_requirement_ir(base),
        'layer_impact': _dq_layer_impact(base),
        'index_bridge': _dq_index_bridge(base, repo_root),
        'final_quality_gate': _dq_final_quality_gate(base),
        'report_quality': _dq_report_quality(base),
        'report_confirmation': _dq_report_confirmation(base),
        'plan_missing_confirmation': _dq_plan_missing_confirmation(base, is_team=is_team),
    }
    if is_team:
        results['team_sub_plans'] = _dq_team_sub_plans(base, member_repos)
        # ── 新增 fan-out/fan-in 检查 ──
        involved_repos = _read_involved_repos(base) or member_repos
        results['per_repo_completeness'] = _dq_per_repo_completeness(base, involved_repos)
        has_contract_changes = _detect_contract_changes(base)
        results['cross_align'] = _dq_cross_align(base, has_contract_changes)
        results['team_section_9'] = _dq_team_section_9(base, involved_repos)
    results['prd_coverage'] = _dq_prd_coverage_simple(base)
    return results
```

- [ ] **Step 3: 在 `detect_team_mode` 函数之后追加两个 helper**

定位：

```python
def detect_team_mode(repo_root):
    ...
```

在它之后（`compute_exit_code` 函数之前）追加：

```python
def _read_involved_repos(base):
    """读取 context/involved-repos.yaml，返回 involved_repos 仓名列表（按 Step 3.5 产出）。

    若文件不存在或解析失败，返回 None；调用者会回落到 member_repos。
    """
    p = base / 'context' / 'involved-repos.yaml'
    if not p.is_file():
        return None
    try:
        data = yaml.safe_load(p.read_text(encoding='utf-8')) or {}
    except Exception:
        return None
    repos = []
    for entry in (data.get('involved_repos') or []):
        repo = entry.get('repo') if isinstance(entry, dict) else None
        if repo:
            repos.append(repo)
    return repos or None


def _detect_contract_changes(base):
    """判断 PRD 是否涉及契约改动。

    若聚合后的 context/contract-delta.yaml 中存在任何 change_type != NO_CHANGE 的 deltas → True。
    无 contract-delta.yaml 或全是 NO_CHANGE → False。
    """
    p = base / 'context' / 'contract-delta.yaml'
    if not p.is_file():
        return False
    try:
        data = yaml.safe_load(p.read_text(encoding='utf-8')) or {}
    except Exception:
        return False
    for d in (data.get('deltas') or []):
        if d.get('change_type', 'NO_CHANGE') != 'NO_CHANGE':
            return True
    return False
```

- [ ] **Step 4: 调用 quality-gate 进行 smoke test**

构造一个最小 fixture，运行 distill 检查：

```bash
python3 << 'EOF'
import importlib.util, tempfile, yaml as y
from pathlib import Path

spec = importlib.util.spec_from_file_location('qg', 'scripts/quality-gate.py')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

with tempfile.TemporaryDirectory() as tmp:
    root = Path(tmp)
    # 团队 profile
    (root / 'team').mkdir()
    (root / 'team' / 'project-profile.yaml').write_text(y.safe_dump({
        'layer': 'team-common',
        'team_repos': [{'repo': 'A'}, {'repo': 'B'}],
    }))
    distill = root / '_prd-tools' / 'distill' / 'demo'
    (distill / '_ingest').mkdir(parents=True)
    (distill / '_ingest' / 'document.md').write_text('# doc')
    (distill / 'report.md').write_text('## §9.1 Frontend\n## §9.2 BFF\n## §9.3 Backend\n## §9.4 External\n## §9.5 跨层\n## §9.A\n## §9.B\n')
    (distill / 'team-plan.md').write_text('plan')
    (distill / 'context').mkdir()
    (distill / 'plans').mkdir()
    (distill / 'plans' / 'plan-A.md').write_text('A')
    (distill / 'plans' / 'plan-B.md').write_text('B')
    # involved-repos
    (distill / 'context' / 'involved-repos.yaml').write_text(y.safe_dump({
        'involved_repos': [{'repo': 'A'}, {'repo': 'B'}],
    }))
    # per-repo for A and B (full)
    for r in ['A', 'B']:
        d = distill / 'per-repo' / r / 'context'
        d.mkdir(parents=True)
        (distill / 'per-repo' / r / 'report.md').write_text('r')
        (d / 'layer-impact.yaml').write_text('x: 1')
        (d / 'contract-delta.yaml').write_text('deltas: []')

    res = mod.run_distill_quality(distill, root)
    assert 'per_repo_completeness' in res, res.keys()
    assert res['per_repo_completeness']['status'] == 'pass', res['per_repo_completeness']
    assert res['team_section_9']['status'] == 'pass', res['team_section_9']
    print('OK')
EOF
```

预期 `OK`。

- [ ] **Step 5: 命令行回归**

```bash
python3 scripts/quality-gate.py distill --help 2>&1 | head -5
```

预期：仍能打印 usage，不因新代码报错。

- [ ] **Step 6: Commit**

```bash
git add scripts/quality-gate.py
git commit -m "feat(quality-gate): integrate fan-out checks into run_distill_quality"
```

---

## Task 8: 端到端验收 + CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`、`plugins/prd-distill/CHANGELOG.md`

- [ ] **Step 1: 检查整体改动清单**

```bash
git log --oneline v2.0..HEAD 2>/dev/null || git log --oneline -20
```

应看到 Task 1-7 的所有 commit。

- [ ] **Step 2: 跑 quality-gate distill 命令检查脚本可用**

```bash
python3 scripts/quality-gate.py distill --help
```

预期：打印 usage，无 ImportError。

- [ ] **Step 3: 检查 markdown 链接（spec / plan 互引）**

```bash
python3 << 'EOF'
import re, os
files = [
    'plugins/prd-distill/skills/team-distill/workflow.md',
    'plugins/prd-distill/skills/team-distill/SKILL.md',
    'plugins/prd-distill/skills/prd-distill/workflow.md',
]
for f in files:
    content = open(f).read()
    # 找 markdown 链接
    for m in re.finditer(r'\[([^\]]+)\]\(([^)]+)\)', content):
        link = m.group(2)
        # 跳过 anchor-only 和 http
        if link.startswith('http') or link.startswith('#'):
            continue
        # 相对路径解析
        target = os.path.normpath(os.path.join(os.path.dirname(f), link.split('#')[0]))
        if not os.path.exists(target):
            print(f'BROKEN: {f} -> {link}')
print('done')
EOF
```

预期：仅打印 `done`，无 `BROKEN` 行（或仅有已知锚点链接）。如有 broken 链接需修。

- [ ] **Step 4: 更新 plugins/prd-distill/CHANGELOG.md**

在文件最顶部 `## [Unreleased]` 节（如不存在则新建）下加：

```markdown
### Added
- **team-distill fan-out/fan-in 编排**：`/team-distill` 重构为「主 agent 编排 + subagent 并行蒸馏 + 主 agent 聚合」两段式架构。subagent 在团队仓内的成员仓 submodule 上跑 prd-distill Step 4-7，主 agent 跨仓对齐契约后聚合 report 与 plan
- 新增 `context/cross-align.yaml` 跨仓对齐产物（schema 见 references/output-contracts.md）
- prd-distill workflow.md 新增「single-repo subagent 模式」小节
- quality-gate 团队模式新增 3 项检查：per-repo 完整性 / cross-align 存在性 / report.md §9 子节齐全

### Changed
- `team_repos[]` schema 新增 `source_path` 与 `submodule` 字段（用户层不破坏兼容：旧 profile 仍可读，仅在跑 fan-out 时校验）
- team-distill SKILL.md 差异表更新：subagent 在 `repos/{repo}/` 内可正常 rg/glob

### Migration
- 团队仓需要将各成员仓加为 git submodule 至 `repos/{repo}/`，并在 `project-profile.yaml` 的 `team_repos[].source_path` 配置路径。未配置时 `/team-distill` 退化为旧"只读 reference"模式（每仓标 unavailable，提示 `git submodule update --init`）。
```

- [ ] **Step 5: 更新根 CHANGELOG.md**

在文件最顶部 `## [Unreleased]` 节加：

```markdown
### Added
- team-distill 升级为 fan-out/fan-in 架构（spec：`docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md`）

### Changed
- `team_repos[]` schema 加 `source_path` / `submodule` 字段
```

- [ ] **Step 6: 验证 CHANGELOG 格式**

```bash
head -30 plugins/prd-distill/CHANGELOG.md
head -20 CHANGELOG.md
```

确认结构合理、字段标题对齐。

- [ ] **Step 7: 最终 Commit**

```bash
git add plugins/prd-distill/CHANGELOG.md CHANGELOG.md
git commit -m "docs: changelog for team-distill fan-out/fan-in"
```

- [ ] **Step 8: 全部改动 diff 复检**

```bash
git diff --stat v2.0..HEAD
git log --oneline v2.0..HEAD
```

预期改动文件清单（约 7 个）：

```
plugins/reference/skills/reference/templates/project-profile.yaml
plugins/reference/skills/team-reference/workflow.md
plugins/prd-distill/skills/prd-distill/workflow.md
plugins/prd-distill/skills/prd-distill/references/output-contracts.md
plugins/prd-distill/skills/team-distill/SKILL.md
plugins/prd-distill/skills/team-distill/workflow.md
scripts/quality-gate.py
plugins/prd-distill/CHANGELOG.md
CHANGELOG.md
```

- [ ] **Step 9: 反膨胀核验**

```bash
# 没新增 skill 文件
git diff --stat v2.0..HEAD -- 'plugins/**/SKILL.md' | grep -v "^ plugins" || echo "OK: 无新增 SKILL.md"
git diff --stat v2.0..HEAD -- 'scripts/*.py' | head -5
# scripts/ 计数不增加
ls scripts/*.py | wc -l
```

预期：scripts/*.py 数量与本 plan 启动前相同（quality-gate.py 行数增加，但文件数不变）。无新增 SKILL.md。

---

## Self-Review

按 plan 规范，把整个文件再过一遍：

- 每个 Task 都有 Files、Steps、Verify、Commit
- 没有 TBD/TODO/"以此类推"
- Python 代码块都含完整函数体
- Markdown 替换段都给出完整新内容

如无问题，进入执行阶段。

---

## 执行注意事项

1. **每个 sub-task 独立 commit**：本 plan 共 13 个 commit（约），便于 PR review
2. **Task 5 (workflow.md 重写) 共 5 个 sub-task，依赖顺序**：5a → 5b → 5c → 5d → 5e
3. **Task 7 (quality-gate) 3 个 sub-task，可顺序**：7a → 7b → 7c
4. **遇到 markdown 链接报错**：参考 Task 8 Step 3 修，禁忌跳过
5. **CLAUDE.md 步骤编号规范的开口子**：本 plan 的 Step 3.5/7.5/7.6 不在内文修 CLAUDE.md；plan 完成后用户决定是否单开 PR 调整规范
