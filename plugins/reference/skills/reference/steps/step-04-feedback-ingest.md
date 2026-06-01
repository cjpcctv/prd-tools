<workflow_state>
  <workflow>reference</workflow>
  <current_step>6</current_step>
  <allowed_inputs>_prd-tools/distill/**/context/reference-update-suggestions.yaml, _prd-tools/reference/, source code</allowed_inputs>
  <must_not_read_by_default>unrelated distill outputs</must_not_read_by_default>
  <must_not_produce>_prd-tools/reference/01-codebase.yaml (without user confirmation)</must_not_produce>
</workflow_state>

## MUST NOT

- MUST verify ALL prerequisite files exist and are non-empty before starting this step
- MUST NOT produce files listed in `<must_not_produce>`
- MUST NOT read files listed in `<must_not_read_by_default>` unless explicitly needed
- MUST NOT proceed if any prerequisite file is missing

# 步骤 6：反馈回流

## 目标

在人工确认后，使用 `/prd-distill` 的输出改进 `_prd-tools/reference/`。

## 输入

- `_prd-tools/distill/**/context/reference-update-suggestions.yaml`
- `_prd-tools/distill/**/report.md`
- 兼容旧版：`_prd-tools/distill/**/spec/reference-update-suggestions.yaml`、`_prd-tools/distill/**/reference-update-suggestions.yaml`、`_prd-tools/distill/**/distilled-report.md`
- 当前 `_prd-tools/reference/`
- 当前源码

## 参数

```yaml
auto_apply_confidence: none | high   # 默认 none（Mode E 行为，全部人工确认）
```

- `none`：所有 suggestion 都进逐条人工确认（即原 Mode E 行为）。
- `high`：满足下方"自动应用 Gate"小节所有 7 条 gate 的 suggestion 直接写入 reference；其余进 `pending_review` 队列。

Mode D 调用本步骤时传 `high`；Mode E 单独运行时保持 `none`。

## 建议类型

- `new_term`
- `new_route`
- `new_contract`
- `new_playbook`
- `golden_sample_candidate`
- `contradiction`

## 执行

1. 收集建议，并按目标 reference 文件分组。
2. 读取 `current_repo_scope`、`owner_to_confirm`、`team_reference_candidate` 和 `team_scope`。
3. 用当前源码或文档重新检查 evidence。
4. 对矛盾项，展示当前 reference 事实、新证据和修复建议。
5. 让用户逐条批准、编辑或跳过。
6. 只应用用户确认过的变更。
7. 更新 `last_verified`。
8. 写入 `_prd-tools/build/feedback-report.yaml`。

`feedback-report.yaml` 的 disposition schema（**4 种** disposition）：

```yaml
applied:           # 已自动写入（仅 auto_apply_confidence=high 时出现）
  - id: SUG-001
    type: new_term
    target: 05-domain.yaml
    auto_applied: true
    gate_passed: [G1, G2, G3, G4, G5, G6, G7]
    evidence: [...]

pending_review:    # 进入逐条人工确认（包括黑名单类型）
  - id: SUG-002
    type: contradiction
    blocked_by: type_blacklist
    notes: "type=contradiction, 永远人工"

rejected:          # 硬性拒绝（极罕见，比如 evidence 完全缺失或 live 验证失败）
  - id: SUG-003
    reason: "no evidence after live verification"

already_present:   # 幂等性：reference 中已有相同 anchor 的事实
  - id: SUG-004
    type: new_term
    target: 05-domain.yaml
    reason: "existing anchor matches; reference already contains this fact"
```

四种 disposition：`applied` / `pending_review` / `rejected` / `already_present`。

## 规则

- 不应用推测性更新。
- 不覆盖无关 reference 内容。
- 不自动删除旧版文件。
- 每个已应用更新都必须有 evidence。
- `current_repo_scope.action: apply_to_current_repo` 且 evidence 可验证时，才允许写入当前仓 confirmed 事实。
- `record_as_signal` 或 `needs_owner_confirmation` 只能写入 handoff、unknowns、owner_to_confirm 或候选字段，不能升级为确定契约。
- `team_reference_candidate: true` 必须保留为候选标记；除非用户明确确认团队治理结果，否则不代表已经同步到团队知识库。

## 自动应用 Gate (auto_apply_confidence=high)

只有同时满足以下 **7 条 gate** 的 suggestion 才会被自动写入 reference；任一条不满足 → 进 `pending_review`，走逐条人工确认。

| # | 条件 | 理由 |
|---|---|---|
| G1 | `type ∈ {new_term, new_playbook, new_route, golden_sample_candidate}` | 纯追加类型，不覆盖任何现有事实 |
| G2 | `confidence == high`（见下方判定） | 低置信度永远进人工 |
| G3 | `current_repo_scope.action == apply_to_current_repo` | 沿用现有规则 |
| G4 | `evidence` 至少 1 条且 **live 验证通过**：引用的源码文件/anchor 当前仍存在；引用的 PRD 段落/anchor 仍能在文档中定位到 | 防止建议生成后源码已变 |
| G5 | 与现有 reference **无 ID/key 冲突**（同名术语、同 endpoint、同 playbook id 都视为冲突） | 冲突 = 潜在矛盾 = 进人工 |
| G6 | `team_reference_candidate != true` | 团队治理永远人工 |
| G7 | `needs_owner_confirmation != true` 且 `record_as_signal != true` | 沿用现有规则 |

### 永远进人工的类型（黑名单）

无论其他 gate 是否通过，以下 type **永远** 不自动应用：

- `contradiction` — Mode E 设计的根本防线
- `new_contract` — 跨团队字段，破坏半径太大

### confidence == high 的判定

`high` = **同时满足**：

- evidence 中至少 1 条源码 anchor（`file_path:line_range`）
- evidence 中至少 1 条 PRD/技术文档 anchor 或 git diff anchor
- PRD 文本中对应描述是直接陈述（不是 "可能" / "或许" / "待定"）
- suggestion 的 fields 全部能从 evidence 直接抽出（不是推断）

否则降为 `medium` 或 `low`。

### 处理顺序

1. **先全量校验** 所有 suggestion 的 7 条 gate 和黑名单
2. 计算每条 suggestion 的 disposition：`applied` / `pending_review` / `rejected` / `already_present`
3. **再批量写入** reference/*.yaml（applied 项）
   - 若批量写入中途失败：已写入项保留，`feedback-report.yaml` 记录失败项为 `rejected`（reason: `write_error`），整体不回滚。
4. 写 `feedback-report.yaml` 记录全部 disposition
5. 输出摘要并询问是否进入 Mode E 处理 pending_review

### 幂等性

重复对同一组 suggestion 跑本步骤：

- step-04 检查 reference 中是否已有相同 anchor 的事实 → 跳过应用，标 `already_present`
- 不会重复增加同名 term / 同 endpoint contract / 同 id playbook

### 摘要呈现

执行完成后给用户的输出格式：

```text
Mode D 完成（或 Mode E auto_apply_confidence=high 完成）。
  自动应用：5 条（3×new_term, 2×new_playbook）
  待人工确认：2 条（1×contradiction, 1×new_contract）
  已存在跳过：1 条
  拒绝：0 条

待确认项已写入 _prd-tools/build/feedback-report.yaml 的 pending_review。
是否现在进入 Mode E 处理这些项？[y/N]
```

用户回 `N`：自动应用部分**保留**（已通过 7 条 gate）；pending_review 项保留待后续单独跑 Mode E 处理。

## Self-Check（回流后必须逐项验证）

> **Self-Check 的两种条目**：本清单同时包含 (a) **机器可验证断言**（标 `[M]`）和 (b) **人工判读提示**（标 `[H]`）。执行 Self-Check 时：
> - `[M]` 条目必须逐条列出 `verify: <命令>` 与 `expect: <结果>`，未通过不得进下一步。
> - `[H]` 条目作为判读提示，LLM 自检后必须写入 workflow-state.yaml 的 `self_check_notes[step_id]` 数组，内容为"我为什么认为这条满足"的简短解释。

- [ ] [M] 每条 suggestion 的 target_file 是存在的 reference 文件
- [ ] [H] apply_to_current_repo 的建议有当前仓源码证据支撑
- [ ] [M] needs_owner_confirmation 的建议填写了 owner_to_confirm
- [ ] [M] golden_sample_candidate 的建议有完整的 lessons 和 evidence
- [ ] [H] 用户确认后才修改 reference，未自动修改
- [ ] [M] 当 auto_apply_confidence=high 时，`applied` 列表中每条的 `gate_passed` 均为 7 元素列表 [G1, G2, G3, G4, G5, G6, G7]（`feedback-report.yaml` 可验证）
