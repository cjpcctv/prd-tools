# Mode D：增量样例补充 — 设计

- 日期：2026-05-31
- 状态：Draft（待 review）
- 影响范围：`plugins/reference/skills/reference/`

## 背景

当前 `/reference` 提供 8 种模式（F/A/B/B2/C/E/T/T2），但**没有**为"已建 reference 后拿到新历史 PRD + 对应变更分支"这个场景设计的入口。

现有 workaround 都不理想：

- 跑 `/prd-distill` 然后 Mode E：会产出无用的 `plan.md`，错配感强
- 重跑 Mode F：会覆盖 `build/context-enrichment.yaml`，且不会自动落到 `reference/*.yaml`
- 直接编辑 `reference/*.yaml`：丢失 cross-sample 模式提取能力

Mode D 填补这个空缺。

## 目标

| 项 | 描述 |
|---|---|
| **G1** | 用户给一个 PRD + 一个 git 分支，工具自动提取知识候选并更新 reference |
| **G2** | 高置信度结果**自动写入** reference，低置信度/有冲突项进 Mode E 人工流程 |
| **G3** | 不破坏现有 reference 已确认事实（git diff 应只见新增） |
| **G4** | 失败可恢复（每条 suggestion 独立事务） |
| **G5** | 不新建任何文件，仅扩展 SKILL.md / workflow.md / step-00 / step-04 |

## 非目标

- ❌ 多分支同时处理（一次一个 PRD+分支）
- ❌ 跨仓样例聚合（属于 Mode T 范畴）
- ❌ medium/low 置信度自动应用（仅 high 自动）
- ❌ 自动应用回滚机制（git 即回滚机制）

## 命令面

### 模式表新增（SKILL.md）

| 模式 | 何时 | 输出 |
|---|---|---|
| D 增量样例补充 | 已建 reference + 拿到新历史 PRD/分支 | 更新后的 `_prd-tools/reference/` + `build/feedback-report.yaml` |

### Phase 映射新增（workflow.md）

| 模式 | 跑哪些 Phase | 跳过 | 关键产物 | 完成判定 |
|---|---|---|---|---|
| **D** 增量样例补充 | Phase 1 (append) → Phase 6 (auto-apply gate) | Phase 2/3/4/5 | `build/context-enrichment.yaml` 追加 + 受影响 `reference/*.yaml` + `build/feedback-report.yaml` | 所有 suggestion 已 dispositioned |

Mode D 沿用现有 [Mode B/B2/C/E 共同规则](../../../plugins/reference/skills/reference/workflow.md)：执行前读 `reference-workflow-state.yaml`、不"顺手补全"、结束更新 state。

## 整体流程

```text
用户 /reference → 选 D → 输入 {prd_path, branch_name, [base_ref?]}
   ↓
前置校验：reference/ 存在 + branch 在当前 repo 可达
   ↓
Phase 1 (append):
  - step-00 --append --branch=<name>
  - git diff <base>..<branch> 拿 files_changed + 内容
  - git log 拿 commit messages 进 evidence
  - 追加新样例到 build/context-enrichment.yaml
   ↓
派生：从新样例生成 reference-update-suggestions.yaml（内部产物）
   ↓
Phase 6 (with auto-apply gate):
  - step-04 --auto-apply-confidence=high
  - 高置信项直接写入 reference/*.yaml
  - 其余项进入现有 Mode E 的逐条确认流程
   ↓
摘要：自动应用 X 条，待确认 Y 条，跳过 Z 条
```

## step-00 扩展（4 处改动）

### S1. workflow_state 放宽

```xml
<must_not_read_by_default>_prd-tools/reference/ (when mode=F; allowed when mode=D)</must_not_read_by_default>
```

### S2. 新增参数

```yaml
mode: F | D            # F=首次构建前，D=已建 reference 后增量补样例
branch: <branch-name>  # Mode D 必填，Mode F 可选
base_ref: <ref>        # 可选，默认推断为 merge-base(branch, default_branch)
```

### S3. 执行流程加 Mode D 分支

每个样例第 2 步追加：

```text
Mode D 时：
  2a. git merge-base <branch> <default_branch>  # 推断 base_ref
      # default_branch 推断顺序：
      #   1. project-profile.yaml 的 default_branch 字段（如有）
      #   2. git symbolic-ref refs/remotes/origin/HEAD（远端 HEAD 指针）
      #   3. fallback: main，再 fallback: master
  2b. git diff <base_ref>..<branch> --stat      # files_changed
  2c. git diff <base_ref>..<branch> -- <files>  # 关键文件 diff 内容
  2d. git log <base_ref>..<branch> --pretty     # commit messages 进 evidence
  2e. files_changed 标 evidence: "git@<branch>:<file>"
```

### S4. 输出加 append 语义

```text
Mode F：覆盖写 build/context-enrichment.yaml
Mode D：
  - 读取现有 build/context-enrichment.yaml
  - 用 (prd_hash, branch) 作为去重 key
  - 已存在的 sample → 整体覆盖那一条
  - 新 sample → append 到 samples[] 并自动分配下一个 SAMPLE-NNN
  - 重算 cross_sample_patterns（基于全量 samples）
  - 更新 collected_at 为当前时间
  - 旧 sample 不动
```

完成判定（"至少 1 个 sample 含 lessons[]"）对 Mode D 只校验**新增**那个样例。

## step-04 扩展（auto-apply gate）

### A1. 新增参数

```yaml
auto_apply_confidence: none | high   # 默认 none（沿用现有 Mode E 行为）
```

Mode D 调用时传 `high`。Mode E 单独运行仍是 `none`。

### A2. 高置信度自动应用：硬性 gate（**全部满足**）

| # | 条件 | 理由 |
|---|---|---|
| G1 | `type ∈ {new_term, new_playbook, new_route, golden_sample_candidate}` | 纯追加类型，不覆盖现有事实 |
| G2 | `confidence == high` | 低置信度永远进人工 |
| G3 | `current_repo_scope.action == apply_to_current_repo` | 沿用现有规则 |
| G4 | `evidence` 至少 1 条且 **live 验证通过**（引用的源码文件/anchor 当前仍存在；引用的 PRD 文档可读） | 防止建议生成后源码已变 |
| G5 | 与现有 reference **无 ID/key 冲突**（同名术语、同 endpoint、同 playbook id） | 冲突 = 潜在矛盾 = 进人工 |
| G6 | `team_reference_candidate != true` | 团队治理永远人工 |
| G7 | `needs_owner_confirmation != true` 且 `record_as_signal != true` | 沿用现有规则 |

任一条不满足 → 进 `pending_review`，走原 Mode E 逐条确认流程。

### A3. 永远进人工的类型（黑名单）

- `contradiction` — Mode E 设计的根本防线
- `new_contract` — 跨团队字段，破坏半径太大

### A4. confidence == high 的判定

```text
high = 同时满足：
  - evidence 中至少 1 条源码 anchor（file_path:line_range）
  - evidence 中至少 1 条 PRD/技术文档 anchor 或 git diff anchor
  - PRD 文本对应描述是直接陈述（不是 "可能"/"或许"/"待定"）
  - suggestion 的 fields 全部能从 evidence 直接抽出（不是推断）

否则降为 medium 或 low。
```

### A5. 输出和审计（feedback-report.yaml）

```yaml
applied:
  - id: SUG-001
    type: new_term
    target: 05-domain.yaml
    auto_applied: true
    gate_passed: [G1, G2, G3, G4, G5, G6, G7]
    evidence: [...]

pending_review:
  - id: SUG-002
    type: contradiction
    blocked_by: G1
    notes: "type=contradiction, 永远人工"

rejected:
  - id: SUG-003
    reason: "no evidence after live verification"

already_present:
  - id: SUG-004
    type: new_term
    target: 05-domain.yaml
    reason: "existing anchor matches; reference already contains this fact"
```

四种 disposition：`applied` / `pending_review` / `rejected` / `already_present`。

### A6. 幂等性

- step-00 用 `(prd_hash, branch)` 去重
- step-04 检查 reference 中已有相同 anchor 的事实 → 跳过应用，标 `already_present`

### A7. 摘要呈现

```text
Mode D 完成。
  自动应用：5 条（3×new_term, 2×new_playbook）
  待人工确认：2 条（1×contradiction, 1×new_contract）
  跳过：1 条（已存在）

待确认项已写入 build/feedback-report.yaml 的 pending_review。
是否现在进入 Mode E 处理这些项？[y/N]
```

## 失败处理

| 失败点 | 副作用范围 | 恢复方式 |
|---|---|---|
| Phase 1 中途崩 | `build/context-enrichment.yaml` 可能有半截 sample | 重跑 Mode D；去重 key 识别并覆盖 |
| Phase 6 写入中途崩 | 已应用的 yaml 已落盘；`feedback-report.yaml` 记录到崩溃前 | 重跑 Mode D；`already_present` 检查跳过已应用 |
| live verification 失败（G4） | 无副作用，suggestion 进 `rejected` | 用户决定是否手动重新生成 |

**核心原则**：每条 suggestion 独立事务，**先全量校验 7 条 gate，再批量写**。

### 用户取消的退出语义

用户在摘要后回 `N`：

- 自动应用部分**保留**（已通过 7 条 gate）
- pending_review 项保留在 `feedback-report.yaml`
- 后续可单独跑 `/reference` Mode E 处理

## 测试策略

### 自动校验层（每次跑完必须通过）

1. `python3 scripts/quality-gate.py reference --root .` 退出码为 0
2. reference/*.yaml git diff 仅见新增（grep 检查无删除行）

### 人工冒烟层（设计落地后做一次）

1. 选项目内一个已合并分支 + 对应历史 PRD
2. 跑 Mode D
3. 检查：
   - `feedback-report.yaml` 的 `applied` 每条都符合 7 条 gate
   - `pending_review` 中无"该自动应用但被卡住"的项
   - reference 文件 diff 仅是新增
4. 冒烟材料归档到 `benchmarks/mode-d-smoke/` 作回归基线

### 不做

- 不引入新的 quality-gate 子命令
- 不引入 mock git 仓库 e2e 测试

## 文档落地点（**0 个新文件**）

- [SKILL.md](../../../plugins/reference/skills/reference/SKILL.md) — 模式表加 D 行
- [workflow.md](../../../plugins/reference/skills/reference/workflow.md) — Phase 映射加 D 行 + 共同规则段加 D
- [step-00-context-enrichment.md](../../../plugins/reference/skills/reference/steps/step-00-context-enrichment.md) — S1-S4 改动
- [step-04-feedback-ingest.md](../../../plugins/reference/skills/reference/steps/step-04-feedback-ingest.md) — A1-A7 改动

预估改动总行数 < 200 行。

## 风险和缓解

| 风险 | 缓解 |
|---|---|
| AI 给自己的 suggestion 盖 high 戳，绕过人工 | A4 双重证据要求 + G4 live 验证 + G5 冲突检测 |
| 自动应用引入与现有事实矛盾的内容 | G5 ID/key 冲突即进人工；A3 黑名单含 contradiction |
| 跨团队契约被自动写 | A3 黑名单含 new_contract；G6 team_reference_candidate 进人工 |
| 重复跑产生脏数据 | A6 幂等性（去重 key + already_present） |

## 反膨胀验证（CLAUDE.md 3 问）

| # | 问题 | 答案 |
|---|---|---|
| 1 | 为什么不能扩展现有文件？ | **正在扩展，不新建**任何文件 |
| 2 | 谁会调用？ | `/reference` Mode D 选项 |
| 3 | 3 个月后没人用会发现吗？ | 会，填补了真实缺口（已有 reference 后增量补样例） |

仅扩展 4 个现有文件，无新增 step / 新增 schema / 新增脚本。
