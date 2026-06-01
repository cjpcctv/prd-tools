# Changelog

> 遵循 [Keep a Changelog](https://keepachangelog.com/) 格式。架构决策详见 [docs/adr/](docs/adr/)。

## [2.20.0] - 2026-06-01

### Added
- feat(quality-gate): integrate fan-out checks into run_distill_quality
- feat(quality-gate): add _dq_cross_align and _dq_team_section_9 checks
- feat(quality-gate): add _dq_per_repo_completeness check
- feat(team-distill): add Step 7.6 report aggregation
- feat(team-distill): add Step 7.5 cross-repo contract alignment
- feat(prd-distill): add single-repo subagent mode for team-distill fan-out
- feat(team-reference): add HEAD consistency check between repos/ and references/
- feat(reference): add source_path/submodule to team_repos schema
- feat(reference): add Mode D for incremental sample enrichment
- feat: SKILL.md + workflow 自动扫描 prd-docs/ 目录作为历史文档输入

### Changed
- chore(contracts): sync output-contracts.md across plugins (cross-align section)
- docs: changelog for team-distill fan-out/fan-in
- docs(prd-distill): add cross-align.yaml schema for team-distill Step 7.5
- refactor(team-distill): rewrite Step 8/9/10/11 for fan-out/fan-in mode
- refactor(team-distill): replace Step 4-7 with subagent fan-out orchestration
- refactor(team-distill): rewrite header + add Step 3.5 (involved repo detection)
- docs(team-distill): refresh SKILL.md frontmatter description for fan-out/fan-in
- docs(team-distill): update SKILL.md to reflect fan-out/fan-in architecture
- docs(superpowers): team-distill fan-out/fan-in implementation plan
- docs(superpowers): team-distill fan-out/fan-in design spec
- docs(superpowers): add Mode D design spec and implementation plan
- Merge branch 'v2.0' of https://github.com/zachary-lz-glm/prd-tools into v2.0
- docs: README 补充 prd-docs 目录准备说明和首次 F-A 流程引导
- 配置与开关

### Fixed
- fix(team-distill): align unavailable_repos shape with output-contracts.md schema
- fix(quality-gate): print fan-out checks + scan per-repo for contract changes
- fix(quality-gate): scope §9 per-repo check to §9 region only
- fix(team-distill): Step 7.5 algorithm — name normalization, layer tie-break, real handoff schema
- fix(team-distill): correct prd-distill workflow.md links + flag Step 3.5 numbering
- fix(prd-distill): correct Step 6/7 labels in subagent skip table
- fix(team-reference): skip HEAD check gracefully when git_head field absent

## [Unreleased]

### Added
- team-distill 升级为 fan-out/fan-in 架构（spec：`docs/superpowers/specs/2026-06-01-team-distill-fanout-design.md`）
- quality-gate.py 团队模式新增 3 项检查（per-repo 完整性 / cross-align 存在性 / §9 子节齐全）

### Changed
- `team_repos[]` schema 加 `source_path` / `submodule` 字段

## [2.19.10] - 2026-05-20

### Changed
- refactor: plan §3 去掉当前代码/目标代码，改为参考文件+注意事项

## [2.19.9] - 2026-05-14

### Fixed
- release.sh --auto 不再覆盖显式版本 + CHANGELOG 插入幂等（修复 awk getline 吞 $0 导致 section 丢失）

## [2.19.8] - 2026-05-14

### Added
- 增强 plan.md §2/§10/§11 — 新增文件树+风险缓解+工作量估算

## [2.19.7] - 2026-05-14

### Added
- prd-distill 支持 .md 文件远程图片 URL 自动下载分析

### Changed
- docs: 清理 README 中已废弃的 AI-friendly PRD / spec 三段式引用
- docs: 更新 README 章节数和描述与 report 9节/plan 12节 对齐
- refactor: report 12→9节业务优先 + plan §2.5映射表 + QA增强

### Fixed
- fix: report 生成前强制等待 context 文件完成，禁止并行写 report

## [2.19.5] - 2026-05-14

### Changed
- refactor: report 12→9节业务优先 + plan §2.5映射表 + QA增强

## [2.19.4] - 2026-05-14

### Added
- feat: 上游接口文档支持 + PRD 项目相关性过滤

### Changed
- refactor: reference 阶段重编 Phase 1-6 + 步骤编号规范
- refactor: 工作流步骤重编 + report §4 需求映射优化
- chore: gitignore install.sh 运行时产物
- refactor: 清理死代码和 afprd 残留
- chore: 删除过时的 .claude/commands/ 目录
- refactor: 删除冗余 schemas/ 和 contracts/ 目录
- refactor: 瘦身 v2.0 — 砍门禁 + 拆团队模式 + 去聚合化 + 恢复 portal
- docs: 团队知识库工作流指南（Mode T/T2 完整流程）

### Fixed
- fix: install.sh 移除已删文件的引用
- fix: 修复 quality-gate.py 运行时崩溃 + 清理孤立文件

- 团队模式补齐 index 能力（多仓 index 加载 + snapshots 同步），聚合策略强化（fatal_cross_layer + endpoint_producer_unverified + 4 条硬约束）

## [2.19.0] - 2026-05-13

- 团队公共知识库聚合与继承（T/T2 模式）
- 自检工具 self-audit，删除 2748 行死代码
- 全盘 audit 修复（gate/workflow/schema 一致性）

## [2.18.1] - 2026-05-12

- Evidence Index 准确性提升（多行签名、跨文件边、增量更新、领域术语桥接）

## [2.18.0] - 2026-05-12

- Artifact Contract 校验框架 + Context Budget + Two-Pass Critic 自检

## [2.17.0] - 2026-05-12

- Workflow State 顺序验证 + Human Checkpoints 脚本化（mode selection / report review）

## [2.16.3] - 2026-05-12

- 保真度优先架构修正 + 三段式工作流 + Step Gate 前置门禁

## [2.16.2] - 2026-05-11

- AI-friendly PRD compiler pipeline

## [2.16.1] - 2026-05-10

- 多层 benchmark + Evidence Index benchmark

## [2.16.0] - 2026-05-08

- prd-distill 支持 .docx 输入 + Claude 多模态图片分析

## [2.15.0] - 2026-05-08

- 重写 README，强化对外可读性

## [2.14.0] - 2026-05-08

- 移除第三方依赖，精简 distill 产出结构

## [2.13.0] - 2026-05-08

- 移除 GitNexus/Graphify 第三方图谱依赖，回归原生能力

## [2.12.0] - 2026-05-08

- 状态面板 MVP + reference 安装流程统一

## [2.11.1] - 2026-05-07

- install.sh / doctor.sh 中文国际化 + 代理检测修复

## [2.11.0] - 2026-05-07

- 输出目录统一为 _prd-tools/ + 插件 README + SKILL.md 流程图

## [2.10.0] - 2026-05-06

- reference 单仓治理 + graph-context 图谱中间层

## [2.9.0] - 2026-05-06

- 输出契约全面升级 + 契约校验自动化

## [2.8.0] - 2026-05-06

- prd-distill 质量复盘：图片 confidence + plan 技术方案升级

## [2.7.0] - 2026-05-06

- 版本迭代自动化（release.sh --auto + post-commit hook）

## [2.6.0] - 2026-05-04

- MarkItDown 集成 + LLM Vision 图片分析 + 多格式支持（pptx/xlsx/html/epub）
- install.sh 完全重写（7 步向导）

## [2.5.1] - 2026-05-01

- 图谱融合端到端补齐

## [2.5.0] - 2026-04-30

- 图谱证据层：GitNexus + Graphify 双图谱集成

## [2.4.1] - 2026-04-29

- 口径一致性修复（版本号/schema_version 统一）

## [2.4.0] - 2026-04-29

- 渐进式披露输出（report 9 章 + plan 可执行手册）

## [2.3.0] - 2026-04-29

- Reference 从 10 文件精简到 6 文件，引入 SSOT

## [2.2.0] - 2026-04-29

- PRD 工程化解析（prd-ingest）+ 读取质量门禁

## [2.1.0] - 2026-04-28

- 能力面适配器 + 安装版本标记

## [2.0.0] - 2026-04-28

- 完整 7 步蒸馏工作流 + Reference v3.1 + 契约差异分析

## [1.1.0] - 2026-04-27

- 确定性事实校验 + 分级质量门控 + 幻觉检测

## [1.0.0] - 2026-04-27

- 初始版本：reference 4 步工作流 + prd-distill 3 步工作流 + 安装脚本
