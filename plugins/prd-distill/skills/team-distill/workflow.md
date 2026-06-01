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
   - 抽 key：`(normalize(deltas[i].name), deltas[i].contract_surface)` —— `normalize` 规则：去首尾空格、统一小写 method、去末尾 `/`、合并多余空白；例：`"POST /api/v1/order/"` → `"post /api/v1/order"`
   - 记录：`{repo, role}`，其中 `role` 推导规则：
     - `deltas[i].producer != repo's layer` → `consumer`
     - `deltas[i].producer == repo's layer`：再校验本仓 reference（`references/{repo}/01-codebase.yaml` 或 `04-routing-playbooks.yaml`）是否声明了对应的 endpoint/route；声明者标 `producer`，未声明者标 `consumer`（同一 layer 多仓时的 tie-break）

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
   读各仓 `references/{repo}/04-routing-playbooks.yaml` 的 `cross_repo_handoffs[]`。每条记录推导：
   - `from_repo` = 该 yaml 文件所属仓（从 `references/{repo}/` 路径取）
   - `to_repo` = 条目的 `repo` 字段
   - `reason` = 条目的 `handoff_reason`
   - 保留 `verification` 与 `owner_to_confirm` 透传

   按 `(from_repo, to_repo, reason)` 三元组去重合并。在 `cross-align.yaml` 的 `handoffs[]` 中输出，每条字段：`{from_repo, to_repo, reason, verification, owner_to_confirm}`。

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
    reason: "下单走 BFF 聚合接口"
    verification: "confirmed"
    owner_to_confirm: ""
unavailable_repos:
  - repo: "dive-be-legacy"
    reason: "submodule_uninitialized"   # 详细 schema 见 output-contracts.md
suspected_missing_fanout: []   # consumer_orphan 推断出的疑似漏 fan-out 仓
```

### 漏判兜底

如有 `consumer_orphan` 且对应 endpoint 在某 `not_involved_repos[]` 仓的 `references/{repo}/03-contracts.yaml` 中声明为 producer → 加入 `suspected_missing_fanout[]`。最终 §9.5 提示用户重跑该仓。

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

检查项见 quality-gate.py 中 `run_distill_quality` 团队模式分支。unavailable 仓不阻断交付，但 gate 输出 `severity: warning` 计入摘要。

