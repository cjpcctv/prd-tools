# team-distill 工作流

<!-- 编号说明：Step 3.5 / 7.5 / 7.6 是显式插入的新流程节点，与 CLAUDE.md "禁止小数编号" 规则的冲突已知；待团队评审后可决定是否单独 PR 调整 CLAUDE.md。 -->

> **架构**：主 agent 编排 + subagent 并行蒸馏 + 主 agent 聚合（fan-out/fan-in）。spec 见 [docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md](../../../../docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md)。
>
> Step 1-3（PRD Ingestion / Evidence / Requirement IR）由**主 agent**执行，与单仓模式相同；详见 [skills/prd-distill/workflow.md](../prd-distill/workflow.md)。
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

PRD Ingestion → Evidence → Requirement IR，流程同 [prd-distill/workflow.md](../prd-distill/workflow.md)。

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

## Step 4：Code Search & Layer Impact（团队模式）

### 4.2 Graph Context（从 Reference 读取）

**禁止执行 rg/glob 命令** — 团队仓没有源码。

对每个 REQ 的扫描流程：

1. 读取各仓 `references/{repo}/01-codebase.yaml` 的模块/枚举/实体，匹配 PRD 相关内容。
2. 对每个 REQ，匹配涉及的仓库和角色（从 03-contracts 的 producer/consumer 关系推断）。
3. 需要契约细节时，读 `references/{repo}/03-contracts.yaml`。
4. 需要路由信息时，读 `references/{repo}/04-routing-playbooks.yaml`。

自动识别涉及仓库：将 PRD requirement 的关键词与各仓的 04-routing-playbooks 和 01-codebase 模块名匹配，确定每个 REQ 涉及哪些仓库及角色（producer/consumer/middleware）。

GCTX entry 标记 `source: "team_reference"`，附带 `repo` 字段。

### 4.3 Layer Impact 生成

4 层 IMP 从各仓 reference 填充。每层的 `code_anchors` 指向对应仓库的 reference 文件路径。

confidence 规则：
- `medium`（默认，未直接验证源码）
- `high`（被多个仓库 reference 交叉验证时）

### 4.5 Context Pack

从 `references/{repo}/index/` 加载多仓 index：

```bash
python3 .prd-tools/scripts/context-pack.py \
  --distill _prd-tools/distill/<slug> \
  --team-references references \
  --out _prd-tools/distill/<slug>/context/context-pack.md
```

## Step 5：Contract Delta（团队模式）

跨仓视角：
- 从各仓 `references/{repo}/03-contracts.yaml` 读取 producer/consumer 信息，构建跨仓契约全景。
- consumer 调用的 endpoint 在其他仓声明为 producer → 标记 cross_repo 契约，`alignment_status: needs_confirmation`。
- 每条 delta 的 `consumers[]` 跨仓填充。

## Step 6-7：Report（团队模式）

report.md §9 强制 5 个子节：
- §9.1 Frontend：前端层 IMP 和契约
- §9.2 BFF：BFF 层 IMP 和契约
- §9.3 Backend：后端层 IMP 和契约
- §9.4 External：外部系统影响
- §9.5 跨层对齐风险：`consumers - checked_by` 不为空 / `alignment_status: blocked` 等

Report Review Gate 同单仓模式。

## Step 8：Plan（团队模式）

生成 `team-plan.md` + N 份 `plans/plan-{repo}.md`。

成员仓列表从 `project-profile.yaml` 的 `team_repos[]` 读取。涉及的仓库和角色从各仓 03-contracts.yaml 自动推断。

**team-plan.md 结构**：
1. **范围与假设**：目标、跨仓依赖、成员仓角色表
2. **涉及仓库总览**：按 repo 分组的代码坐标、跨仓调用链、关键设计决策
3. **跨仓时序**：Phase 1-N 跨仓依赖图、每个仓的交付里程碑
4. **Sub-Plan 索引表**：列出所有 sub-plan 文件名 + 对应仓 + IMP 数
5. **契约对齐（跨仓）**：从 contract-delta.yaml 提取跨仓契约摘要
6. **风险与回滚**：跨仓联调风险、回滚策略
7. **工作量总览**：按仓汇总

**plans/plan-{repo}.md**：复用标准 11-section plan 模板，scope 限定到单个成员仓。

文件名从 `team_repos[].repo` 动态生成，禁止硬编码。

## Step 9-11：同单仓

Readiness Score、Reference Backflow、Quality Gate 流程同单仓模式。

Quality Gate 团队模式检查 `team-plan.md` + `plans/` 目录（而非 `plan.md`）。

团队模式 Reference 回流额外触发条件：
- 发现跨仓契约、owner、handoff 或团队级术语候选，但当前仓不能独立确认。
- `team_reference_candidate: true` 标记为团队知识库收集候选。
