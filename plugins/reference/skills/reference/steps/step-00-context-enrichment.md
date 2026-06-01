<workflow_state>
  <workflow>reference</workflow>
  <current_step>1</current_step>
  <allowed_inputs>user-provided PRD paths, historical samples</allowed_inputs>
  <must_not_read_by_default>_prd-tools/reference/ (when mode=F; allowed when mode=D)</must_not_read_by_default>
  <must_not_produce>_prd-tools/reference/01-codebase.yaml</must_not_produce>
</workflow_state>

## MUST NOT

- MUST verify ALL prerequisite files exist and are non-empty before starting this step
- MUST NOT produce files listed in `<must_not_produce>`
- MUST NOT read files listed in `<must_not_read_by_default>` unless explicitly needed
- MUST NOT proceed if any prerequisite file is missing

# 步骤 1：上下文收集

## 目标

在构建 reference v4.0 之前，从历史 PRD、技术方案、分支和 diff 中提取可复用项目知识。

本步骤只收集事实，不修改 `_prd-tools/reference/`。

## 自动发现

执行前先扫描项目根目录下的 `prd-docs/`：

1. 检查 `<项目根>/prd-docs/` 是否存在。
2. 如存在，自动收集其中所有 `.md`、`.docx`、`.txt` 文件作为历史文档候选，按文件名或标题分组为样例。
3. 如不存在或文件不足，向用户收集补充材料（手动提供路径或粘贴）。
4. 将自动发现的文件展示给用户确认，用户可以增删或替换。

## 参数

```yaml
mode: F | D            # F=首次构建前（默认），D=已建 reference 后增量补样例
branch: <branch-name>  # Mode D 必填，Mode F 可选
base_ref: <ref>        # 可选；缺省时按以下顺序推断：
                       #   1. project-profile.yaml 的 default_branch 字段
                       #   2. git symbolic-ref refs/remotes/origin/HEAD
                       #   3. fallback: main → master
```

## 输入

优先使用 `prd-docs/` 中的自动发现文件，不足时向用户收集 1-3 组历史样例：

- PRD 路径
- 可选技术方案/API 文档路径
- 前端代码库路径和分支/commit
- BFF 代码库路径和分支/commit
- 后端代码库路径和分支/commit
- 如有，补充已知事故、回滚、返工说明

## 执行

对每个样例：

1. 读取 PRD 和技术文档。
2. 在用户提供的 repo 范围内检查 git branch/diff。

   Mode D 时，步骤 2 替换为：
   - 2a. `git merge-base <branch> <default_branch>` 推断 base_ref（如未显式提供；推断顺序见上方 `## 参数` 的 `base_ref` 注释）
   - 2b. `git diff <base_ref>..<branch> --stat` 拿 files_changed
   - 2c. `git diff <base_ref>..<branch> -- <files>` 拿关键文件 diff 内容
   - 2d. `git log <base_ref>..<branch> --pretty` 拿 commit messages 进 evidence
   - 2e. files_changed 中每条标 `evidence: "git@<branch>:<file>"`

3. 将 PRD 描述映射到实际变更文件和契约面。
4. 提取术语、路由信号、契约面、playbook 步骤、QA 用例、坑点和高风险文件。
5. 不确定就记录不确定，不猜测。

## 输出

写入 `_prd-tools/build/context-enrichment.yaml`：

```yaml
schema_version: "4.0"
tool_version: "<tool-version>"
collected_at: ""
samples:
  - id: "SAMPLE-001"
    title: ""
    docs: []
    repos:
      frontend: { path: "", branch: "" }
      bff: { path: "", branch: "" }
      backend: { path: "", branch: "" }
    requirement_signals: []
    files_changed: []
    contract_surfaces: []
    playbook_candidates: []
    glossary_candidates: []
    pitfalls: []
    qa_cases: []
    evidence: []
cross_sample_patterns:
  routing: []
  contracts: []
  playbooks: []
  risks: []
```

### 写入语义

- **Mode F**：覆盖写 `_prd-tools/build/context-enrichment.yaml`。
- **Mode D**：
  1. 读取现有 `_prd-tools/build/context-enrichment.yaml`。
  2. 用 `(prd_path_hash, branch)` 作为去重 key（`prd_path_hash` = hash of sorted `prd_paths` values in the sample）。
  3. 已存在的 sample → 整体覆盖那一条；新 sample → append 到 `samples[]` 并自动分配下一个 `SAMPLE-NNN`。
  4. 重算 `cross_sample_patterns`（基于全量 samples）。
  5. 更新 `collected_at` 为当前时间。
  6. 旧 sample 不动。

完成判定（"至少 1 个 sample 含 lessons[]"）对 Mode D 只校验**新增的那个样例**。

## 映射到 Reference v4.0

- 术语候选 -> `05-domain.yaml`
- 路由信号 -> `04-routing-playbooks.yaml`
- 契约面 -> `03-contracts.yaml`
- playbook、坑点、QA 用例、golden sample -> `04-routing-playbooks.yaml`
- 高风险文件 -> `02-coding-rules.yaml`（danger_zones）
- 业务决策 -> `05-domain.yaml`（decision_log）
- 枚举、结构体、模块 -> `01-codebase.yaml`
- 编码规范和约束 -> `02-coding-rules.yaml`
