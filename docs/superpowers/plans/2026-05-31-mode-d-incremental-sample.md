# Mode D 增量样例补充 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `/reference` 增加 Mode D，让用户提供"新历史 PRD + 对应变更分支"后，工具自动收集上下文并将高置信度知识写入 `_prd-tools/reference/`，其余进 Mode E 人工确认流程。

**Architecture:** 不新建任何文件。仅扩展 4 个现有文件（SKILL.md / workflow.md / step-00 / step-04）。Mode D 由 Phase 1 (append) + Phase 6 (auto-apply gate) 组合而成，复用 Mode F 的样例提取能力和 Mode E 的回流通道。

**Tech Stack:** Markdown (skill 定义 + workflow.md + step 文件)；bash + git (用于 step-00 的 diff/log 抽取)；Python `scripts/quality-gate.py` (验收)。

**关键参考：**
- 设计：[docs/superpowers/specs/2026-05-31-mode-d-incremental-sample-design.md](../specs/2026-05-31-mode-d-incremental-sample-design.md)
- 项目反膨胀规则：[CLAUDE.md](../../../CLAUDE.md)
- 现有模式定义：[plugins/reference/skills/reference/SKILL.md](../../../plugins/reference/skills/reference/SKILL.md)
- Phase 映射：[plugins/reference/skills/reference/workflow.md](../../../plugins/reference/skills/reference/workflow.md)

**Commit 策略：** 全部改动合并为 1 个 `feat:` commit（符合 [acac9f2](https://github.com/cjpcctv/prd-tools) 的现有 pattern：SKILL+workflow 联合更新）。中间任务**只 stage 不 commit**，最后一个任务统一提交，避免多次触发 post-commit 自动 release。

---

### Task 1：SKILL.md 增加 Mode D 行

**Files:**
- Modify: `plugins/reference/skills/reference/SKILL.md:28-36`

- [ ] **Step 1：写失败的一致性检查**

```bash
# 在 SKILL.md 中应能找到 Mode D 行；现在还没有，应该返回 0 行
grep -c "^| D 增量样例补充" plugins/reference/skills/reference/SKILL.md
```

Expected: `0`

- [ ] **Step 2：在模式表追加 D 行**

把 SKILL.md 第 35 行（`| E 反馈回流 ...`）下面追加：

```markdown
| D 增量样例补充 | 已建 reference + 拿到新历史 PRD/分支 | 更新后的 `_prd-tools/reference/` + `_prd-tools/build/feedback-report.yaml` |
```

注意保持表格列对齐和分隔符 `|` 数量一致（3 列）。

- [ ] **Step 3：验证一致性检查通过**

```bash
grep -c "^| D 增量样例补充" plugins/reference/skills/reference/SKILL.md
```

Expected: `1`

- [ ] **Step 4：stage 改动（不 commit）**

```bash
git add plugins/reference/skills/reference/SKILL.md
git status --short
```

Expected: `M  plugins/reference/skills/reference/SKILL.md`

---

### Task 2：workflow.md 增加 Mode D Phase 映射 + 共同规则更新

**Files:**
- Modify: `plugins/reference/skills/reference/workflow.md:262-277`

- [ ] **Step 1：写失败的一致性检查**

```bash
grep -c "^\| \*\*D\*\* 增量样例补充" plugins/reference/skills/reference/workflow.md
```

Expected: `0`

- [ ] **Step 2：标题改为兼容 D**

把第 258 行：

```markdown
## Mode → Phase 映射（B / B2 / C / E 可执行清单）
```

改为：

```markdown
## Mode → Phase 映射（B / B2 / C / D / E 可执行清单）
```

- [ ] **Step 3：在 Phase 映射表追加 D 行**

在第 269 行（E 反馈回流行）下面追加：

```markdown
| **D** 增量样例补充 | Phase 1（--append --branch=<name>）→ Phase 6（--auto-apply-confidence=high） | Phase 2/3/4/5 | `build/context-enrichment.yaml`（追加新样例）+ 受影响 `reference/*.yaml` + `build/feedback-report.yaml` | 所有 suggestion 已 dispositioned (applied / pending_review / rejected / already_present) |
```

- [ ] **Step 4：共同规则段扩到 D**

把第 273 行：

```markdown
**Mode B/B2/C/E 共同规则**：
```

改为：

```markdown
**Mode B/B2/C/D/E 共同规则**：
```

- [ ] **Step 5：验证 3 处一致性**

```bash
grep -c "B / B2 / C / D / E" plugins/reference/skills/reference/workflow.md \
  && grep -c "^\| \*\*D\*\* 增量样例补充" plugins/reference/skills/reference/workflow.md \
  && grep -c "Mode B/B2/C/D/E 共同规则" plugins/reference/skills/reference/workflow.md
```

Expected: 三个 grep 都返回 `1`

- [ ] **Step 6：stage 改动**

```bash
git add plugins/reference/skills/reference/workflow.md
```

---

### Task 3：step-00 加 Mode D 支持（S1-S4）

**Files:**
- Modify: `plugins/reference/skills/reference/steps/step-00-context-enrichment.md`

- [ ] **Step 1：写失败的一致性检查**

```bash
grep -c "mode=D" plugins/reference/skills/reference/steps/step-00-context-enrichment.md
```

Expected: `0`

- [ ] **Step 2：S1 — workflow_state 放宽**

把第 5 行：

```xml
<must_not_read_by_default>_prd-tools/reference/ (does not exist yet)</must_not_read_by_default>
```

改为：

```xml
<must_not_read_by_default>_prd-tools/reference/ (when mode=F; allowed when mode=D)</must_not_read_by_default>
```

- [ ] **Step 3：S2 — 在 "## 输入" 节前插入 "## 参数" 节**

在 "## 输入" 标题（约第 33 行）**前**插入：

```markdown
## 参数

```yaml
mode: F | D            # F=首次构建前（默认），D=已建 reference 后增量补样例
branch: <branch-name>  # Mode D 必填，Mode F 可选
base_ref: <ref>        # 可选；缺省时按以下顺序推断：
                       #   1. project-profile.yaml 的 default_branch 字段
                       #   2. git symbolic-ref refs/remotes/origin/HEAD
                       #   3. fallback: main → master
```

```

注意：上述代码块用三个反引号包围，外层 markdown 渲染需要逃逸；写入文件时直接写实际反引号即可，不要加额外转义。

- [ ] **Step 4：S3 — 在 "## 执行" 节的步骤 2 之后追加 Mode D 分支**

找到 "## 执行" 节中"2. 在用户提供的 repo 范围内检查 git branch/diff。"这一行（约第 49 行），在它**下面**插入：

```markdown

   Mode D 时，步骤 2 替换为：
   - 2a. `git merge-base <branch> <default_branch>` 推断 base_ref（如未显式提供）
   - 2b. `git diff <base_ref>..<branch> --stat` 拿 files_changed
   - 2c. `git diff <base_ref>..<branch> -- <files>` 拿关键文件 diff 内容
   - 2d. `git log <base_ref>..<branch> --pretty` 拿 commit messages 进 evidence
   - 2e. files_changed 中每条标 `evidence: "git@<branch>:<file>"`
```

- [ ] **Step 5：S4 — 在 "## 输出" 节追加 append 语义**

在 yaml 代码块（约第 83 行 `evidence: []` 下面的 `cross_sample_patterns` 块结束后）**之后**追加：

```markdown

### 写入语义

- **Mode F**：覆盖写 `_prd-tools/build/context-enrichment.yaml`。
- **Mode D**：
  1. 读取现有 `_prd-tools/build/context-enrichment.yaml`。
  2. 用 `(prd_path_hash, branch)` 作为去重 key。
  3. 已存在的 sample → 整体覆盖那一条；新 sample → append 到 `samples[]` 并自动分配下一个 `SAMPLE-NNN`。
  4. 重算 `cross_sample_patterns`（基于全量 samples）。
  5. 更新 `collected_at` 为当前时间。
  6. 旧 sample 不动。

完成判定（"至少 1 个 sample 含 lessons[]"）对 Mode D 只校验**新增的那个样例**。
```

- [ ] **Step 6：验证 4 处一致性**

```bash
grep -c "mode=D" plugins/reference/skills/reference/steps/step-00-context-enrichment.md \
  && grep -c "^## 参数" plugins/reference/skills/reference/steps/step-00-context-enrichment.md \
  && grep -c "Mode D 时" plugins/reference/skills/reference/steps/step-00-context-enrichment.md \
  && grep -c "### 写入语义" plugins/reference/skills/reference/steps/step-00-context-enrichment.md
```

Expected: 全部返回 `1` 或更大

- [ ] **Step 7：stage 改动**

```bash
git add plugins/reference/skills/reference/steps/step-00-context-enrichment.md
```

---

### Task 4：step-04 加 auto-apply gate（A1-A7）

**Files:**
- Modify: `plugins/reference/skills/reference/steps/step-04-feedback-ingest.md`

- [ ] **Step 1：写失败的一致性检查**

```bash
grep -c "auto_apply_confidence" plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
```

Expected: `0`

- [ ] **Step 2：A1 — 在 "## 输入" 节后插入 "## 参数" 节**

找到 "## 输入" 节末尾（约第 28 行 `当前源码` 之后），在 "## 建议类型" 标题**前**插入：

```markdown
## 参数

```yaml
auto_apply_confidence: none | high   # 默认 none（Mode E 行为，全部人工确认）
```

- `none`：所有 suggestion 都进逐条人工确认（即原 Mode E 行为）。
- `high`：满足 §A2 所有 7 条 gate 的 suggestion 直接写入 reference；其余进 `pending_review` 队列。

Mode D 调用本步骤时传 `high`；Mode E 单独运行时保持 `none`。

```

- [ ] **Step 3：A2 + A3 — 在 "## 规则" 节后插入新节 "## 自动应用 Gate (auto_apply_confidence=high)"**

在 "## 规则" 节末尾（约第 58 行）后、"## Self-Check" 标题**前**插入：

```markdown
## 自动应用 Gate (auto_apply_confidence=high)

只有同时满足以下 **7 条 gate** 的 suggestion 才会被自动写入 reference；任一条不满足 → 进 `pending_review`，走逐条人工确认。

| # | 条件 | 理由 |
|---|---|---|
| G1 | `type ∈ {new_term, new_playbook, new_route, golden_sample_candidate}` | 纯追加类型，不覆盖任何现有事实 |
| G2 | `confidence == high`（见下方判定） | 低置信度永远进人工 |
| G3 | `current_repo_scope.action == apply_to_current_repo` | 沿用现有规则 |
| G4 | `evidence` 至少 1 条且 **live 验证通过**：引用的源码文件/anchor 当前仍存在；引用的 PRD 文档可读 | 防止建议生成后源码已变 |
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
4. 写 `feedback-report.yaml` 记录全部 disposition
5. 输出摘要并询问是否进入 Mode E 处理 pending_review

```

- [ ] **Step 4：A5 — 修改输出节（feedback-report.yaml schema）**

找到 "## 执行" 节第 8 步 `8. 写入 _prd-tools/build/feedback-report.yaml。`（约第 48 行），在它**下面**插入：

```markdown

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
    blocked_by: G1
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

```

- [ ] **Step 5：A6 — 在新插入的 "## 自动应用 Gate" 节末尾追加幂等性说明**

在 §A2/A3/A4 内容之后、Self-Check 节之前补一段：

```markdown
### 幂等性

重复对同一组 suggestion 跑本步骤：

- step-04 检查 reference 中是否已有相同 anchor 的事实 → 跳过应用，标 `already_present`
- 不会重复增加同名 term / 同 endpoint contract / 同 id playbook
```

- [ ] **Step 6：A7 — 在该节最后追加摘要呈现示例**

在 §A6 幂等性之后追加：

```markdown
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

```

- [ ] **Step 7：在 Self-Check 增加 1 条机器可验证项**

在 Self-Check 节末尾（最后一个 `[H]` 条目下面）增加：

```markdown
- [ ] [M] 当 auto_apply_confidence=high 时，applied 列表中的每条都通过了 G1-G7 全部 gate（feedback-report.yaml 的 gate_passed 字段为 7 元素列表）
```

- [ ] **Step 8：验证 6 处一致性**

```bash
F=plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
grep -c "auto_apply_confidence" "$F" \
  && grep -c "## 自动应用 Gate" "$F" \
  && grep -c "G1.*new_term" "$F" \
  && grep -c "永远进人工的类型" "$F" \
  && grep -c "already_present" "$F" \
  && grep -c "### 幂等性" "$F"
```

Expected: 全部返回 `1` 或更大

- [ ] **Step 9：stage 改动**

```bash
git add plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
```

---

### Task 5：跨文件一致性 + quality-gate 验收

**Files:** （只读，不修改）

- [ ] **Step 1：跨文件交叉引用检查**

```bash
# Mode D 应在所有 4 个文件出现
for f in \
  plugins/reference/skills/reference/SKILL.md \
  plugins/reference/skills/reference/workflow.md \
  plugins/reference/skills/reference/steps/step-00-context-enrichment.md \
  plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
do
  echo "=== $f ==="
  grep -c "Mode D\|mode=D\|^| D 增量\|\*\*D\*\* 增量" "$f"
done
```

Expected: 每个文件至少返回 1（具体值因文件不同而异，但都 ≥1）

- [ ] **Step 2：参数命名一致性**

```bash
# auto_apply_confidence 应在 step-04 和 workflow.md 都有
grep -l "auto_apply_confidence" \
  plugins/reference/skills/reference/workflow.md \
  plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
```

Expected: 输出两个文件路径

```bash
# --branch 参数应在 step-00 和 workflow.md 都有
grep -l "\-\-branch" \
  plugins/reference/skills/reference/workflow.md \
  plugins/reference/skills/reference/steps/step-00-context-enrichment.md
```

Expected: 输出两个文件路径

- [ ] **Step 3：disposition 4 种取值都被定义**

```bash
F=plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
for d in applied pending_review rejected already_present; do
  echo -n "$d: "
  grep -c "^$d:" "$F"
done
```

Expected: 每行后面是 `1`（在 schema 代码块中各出现一次作为 yaml key）

- [ ] **Step 4：运行 quality-gate.py reference**

> 注意：本仓库自身没有 `_prd-tools/reference/`（这是个 meta 仓库），quality-gate 设计是在**安装到目标项目**后运行。所以这一步应该在已用 PRD Tools 构建过 reference 的目标项目里跑，不在本仓库跑。

如果本仓库有 `_prd-tools/reference/`（之前用工具自构建过），跑：

```bash
python3 scripts/quality-gate.py reference --root . 2>&1
echo "exit: $?"
```

Expected: `exit: 0`

如果没有，跳过此步骤，记录在下一步的 commit message 里。

- [ ] **Step 5：人工通读检查**

打开 4 个文件，确认：

1. SKILL.md 模式表 D 行格式与其他行一致（3 列、对齐符号）
2. workflow.md Phase 映射表 D 行格式与其他行一致（5 列）
3. step-00 的 "## 参数" 节、Mode D 分支、写入语义三处文字风格与原文一致
4. step-04 的"自动应用 Gate"节内 G1-G7 编号清晰，黑名单和判定标准都可索引

发现问题 → 返回对应 Task 修正；通过 → 进入 Task 6。

---

### Task 6：单一 feat commit 提交

**Files:** （之前已 stage 的 4 个文件）

- [ ] **Step 1：检查暂存区**

```bash
git status --short
```

Expected: 4 个 `M` 行：

```
M  plugins/reference/skills/reference/SKILL.md
M  plugins/reference/skills/reference/workflow.md
M  plugins/reference/skills/reference/steps/step-00-context-enrichment.md
M  plugins/reference/skills/reference/steps/step-04-feedback-ingest.md
```

如果还有未 stage 的相关改动，用 `git add` 补上。

- [ ] **Step 2：再次 diff 确认**

```bash
git diff --cached --stat
```

Expected: 4 个文件，总改动行数 < 250（设计目标 < 200，留 25% buffer）

如果改动行数远超 250 → 检查是否有非必要扩展，回退后只保留设计文档列出的内容。

- [ ] **Step 3：创建 feat commit**

```bash
git commit -m "$(cat <<'EOF'
feat(reference): add Mode D for incremental sample enrichment

新增 /reference Mode D：用户提供 PRD 路径 + 变更分支后，自动跑
Phase 1 (--append) 收集新样例，再跑 Phase 6 (--auto-apply-confidence=high)
将高置信度知识写入 reference，其余进 Mode E 人工确认。

改动：
- SKILL.md: 模式表加 D 行
- workflow.md: Phase 映射加 D 行 + 共同规则段扩到 D
- step-00: 加 mode/branch/base_ref 参数 + git diff/log 抽取分支
        + append 写入语义（去重 key 为 prd_hash+branch）
- step-04: 加 auto_apply_confidence 参数 + 7 条 gate (G1-G7)
        + 黑名单 (contradiction, new_contract)
        + 4 种 disposition (applied/pending_review/rejected/already_present)
        + 幂等性说明

不新建任何文件；改动总行数 < 250。

设计文档：docs/superpowers/specs/2026-05-31-mode-d-incremental-sample-design.md

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 4：验证 commit 落地**

```bash
git log -1 --stat
```

Expected:
- commit message 含 `feat(reference): add Mode D`
- 4 个文件出现在 stat 中
- 没有未预期的文件

- [ ] **Step 5：观察 post-commit 自动 release（如启用）**

CLAUDE.md 描述 `feat:` 会触发 minor 版本号 bump（例如 2.19.10 → 2.20.0），五处 VERSION 同步。

```bash
cat VERSION
git log -2 --oneline
```

Expected:
- 如果 hook 已安装：`VERSION` 已更新；最新 commit 形如 `chore: release v2.20.0`
- 如果未安装：保持原版本号；只有刚才那个 feat commit

如果用户**不希望**这次提交触发 release，应该在 Step 3 改用：

```bash
PRD_TOOLS_NO_AUTO_RELEASE=1 git commit -m "..."
```

并手动用 `scripts/release.sh minor` 在合适时机统一发版。

---

### Task 7：交付摘要 + 冒烟测试 runbook

**Files:** （不修改文件，输出给用户）

- [ ] **Step 1：生成交付摘要**

向用户报告：

```text
Mode D 实现完成。
  改动文件：4 个
  改动行数：<diff stat 数字>
  新建文件：0 个
  commit：<commit hash> (feat: add Mode D)
  版本：<VERSION 内容>（如有自动 bump 则标明前后版本）

下一步建议（人工冒烟）：
1. 在已用 PRD Tools 构建过 reference 的目标项目里
2. 选一个已合并的 feature branch + 对应历史 PRD
3. 跑 /reference → 选 D → 输入 PRD 路径和分支名
4. 检查 _prd-tools/build/feedback-report.yaml：
   - applied 中每条都有 gate_passed: [G1..G7]
   - pending_review 中没有"该自动应用但被卡住"的项
   - reference/*.yaml 的 git diff 仅是新增（无删除/修改行）
5. 跑 python3 scripts/quality-gate.py reference --root .，确认退出码 0
```

- [ ] **Step 2：把冒烟结果归档（可选）**

如果用户当场跑了冒烟测试且通过：

```bash
mkdir -p benchmarks/mode-d-smoke/
# 把脱敏后的 PRD 副本、变更分支名、feedback-report.yaml 拷过来作回归基线
```

> ⚠️ 这一步是 **可选** 的，符合 CLAUDE.md 反膨胀第 1 问"为什么不能扩展现有？"——只有当确实跑了冒烟测试并希望保留基线时才建。否则跳过。

---

## Self-Review 结果

### 1. Spec 覆盖

逐项对照 [spec](../specs/2026-05-31-mode-d-incremental-sample-design.md)：

| Spec 节 | 实现 Task |
|---|---|
| 模式表 (SKILL.md) | Task 1 |
| Phase 映射 + 共同规则 (workflow.md) | Task 2 |
| step-00 S1-S4 | Task 3 (Step 2-5) |
| step-04 A1 (参数) | Task 4 (Step 2) |
| step-04 A2 (7 条 gate) + A3 (黑名单) + A4 (high 判定) | Task 4 (Step 3) |
| step-04 A5 (4 种 disposition schema) | Task 4 (Step 4) |
| step-04 A6 (幂等性) | Task 4 (Step 5) |
| step-04 A7 (摘要呈现) | Task 4 (Step 6) |
| 失败处理（事务边界） | Task 4 Step 3 中 "处理顺序"小节（先校验后批量写） |
| 测试策略：自动校验 | Task 5 (quality-gate) |
| 测试策略：人工冒烟 | Task 7 |
| 风险缓解（双重证据 / 冲突检测 / 黑名单） | Task 4 Step 3 |
| 反膨胀验证（0 新文件） | Task 6 Step 2 (改动行数 < 250 上限) |

**无遗漏。**

### 2. 占位符扫描

✅ 无 TODO / TBD / "实现细节稍后填写"
✅ 每一步都有具体命令和预期输出
✅ 所有插入的 markdown 文本都是字面内容，可直接复制

### 3. 类型/命名一致性

- `auto_apply_confidence`: Task 2/4 ✅
- `--branch`: Task 2/3 ✅
- `--append`: Task 2/3 ✅
- `--auto-apply-confidence=high`: Task 2/4 ✅
- `(prd_path_hash, branch)` 去重 key: Task 3 与 spec §S4 ✅
- 4 种 disposition (`applied` / `pending_review` / `rejected` / `already_present`): Task 2/4 ✅
- G1-G7 编号: Task 4/Task 5/spec ✅

**全部对齐。**
