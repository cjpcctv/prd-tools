# team-reference 工作流

> **定位**：将各成员仓库的 reference 原样收集到团队仓库，供团队级 PRD 蒸馏使用。不做聚合、合并或冲突检测。

## Mode T：收集

**前置条件**：

- 团队仓库含 `project-profile.yaml`，配置了 `team_repos[]`
- 各成员仓库已运行 `/reference` Mode A/B，有完整 01-05 YAML + index
- 各成员仓库已将 `_prd-tools/reference/` 提交到 git

**执行步骤**：

1. **读取配置**：`project-profile.yaml` 的 `team_repos[]`，获取成员仓库名和本地路径
2. **逐个收集**：对每个成员仓库：
   - 验证 `local_path/_prd-tools/reference/` 目录存在
   - 将整个 `reference/` 目录内容复制到 `references/{repo-name}/`
   - 包括 01-05 YAML、`project-profile.yaml`、`index/` 目录
3. **HEAD 一致性校验**（团队仓含 submodule 时）：
   - 若 `<团队仓>/repos/{repo}/` 存在（submodule 已 init），运行 `git -C repos/{repo} rev-parse HEAD` 取实际 source HEAD
   - 与 `references/{repo}/project-profile.yaml` 的 `git_head` 字段比对
   - **不一致只发警告，不阻断收集**：把该仓加入 `head_mismatch[]` 摘要字段
   - 团队仓没有 `repos/` 或某仓未 init submodule 时跳过此校验
4. **输出摘要**：哪些仓库收集成功、哪些跳过（路径不存在或 reference 不完整）、哪些 HEAD 不一致（`head_mismatch[]`）

**目录结构**：

```text
references/
  {repo-name}/
    01-codebase.yaml
    02-coding-rules.yaml
    03-contracts.yaml
    04-routing-playbooks.yaml
    05-domain.yaml
    project-profile.yaml
    index/
      entities.json
      edges.json
      inverted-index.json
      manifest.yaml
```

各仓库的 reference 保持独立，不做任何合并或转换。

**硬约束**：

1. **原样复制**：不修改、不合并、不过滤成员仓库的 reference 内容
2. **增量更新**：每次收集覆盖已有目录，保持与成员仓库最新状态一致
3. **跳过不可达仓库**：`local_path` 不存在时跳过并在摘要中标记，不中断整体收集
4. **不主动 submodule update**：team-reference 不替用户跑 `git submodule update`，源码同步是用户责任；本步骤只做"提示"，不擅自动 git。
