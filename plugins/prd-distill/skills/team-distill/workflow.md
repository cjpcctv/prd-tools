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

详细 schema 见 [prd-distill/references/output-contracts.md](../prd-distill/references/output-contracts.md) 的 `context/cross-align.yaml` 章节。简要结构：

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
