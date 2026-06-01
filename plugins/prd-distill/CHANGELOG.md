# Changelog

All notable changes to the **prd-distill** plugin are documented here.

## [Unreleased]

### Added
- **team-distill fan-out/fan-in 编排**：`/team-distill` 重构为「主 agent 编排 + subagent 并行蒸馏 + 主 agent 聚合」两段式架构。subagent 在团队仓内的成员仓 submodule 上跑 prd-distill Step 4-7，主 agent 跨仓对齐契约后聚合 report 与 plan
- 新增 `context/cross-align.yaml` 跨仓对齐产物（schema 见 references/output-contracts.md）
- prd-distill workflow.md 新增「single-repo subagent 模式」小节
- quality-gate 团队模式新增 3 项检查：per-repo 完整性 / cross-align 存在性 / report.md §9 子节齐全
- Step 3.5 涉及仓识别 / Step 7.5 跨仓对齐 / Step 7.6 Report 聚合（主 agent）

### Changed
- `team_repos[]` schema 新增 `source_path` 与 `submodule` 字段（向后兼容：旧 profile 仍可读，仅在跑 fan-out 时校验）
- team-distill SKILL.md 差异表更新：subagent 在 `repos/{repo}/` 内可正常 rg/glob

### Migration
- 团队仓需要将各成员仓加为 git submodule 至 `repos/{repo}/`，并在 `project-profile.yaml` 的 `team_repos[].source_path` 配置路径。未配置时 `/team-distill` 退化为旧"只读 reference"模式（每仓标 unavailable，提示 `git submodule update --init`）。

## [2.19.10] - 2026-05-20

### Changed
- refactor: plan §3 去掉当前代码/目标代码，改为参考文件+注意事项

## [2.19.9] - 2026-05-14

### Fixed
- release.sh --auto 不再覆盖显式版本 + CHANGELOG 插入幂等

## [2.19.8] - 2026-05-14

### Added
- 增强 plan.md §2/§10/§11 — 新增文件树+风险缓解+工作量估算

## [2.19.7] - 2026-05-14

### Added
- prd-distill 支持 .md 文件远程图片 URL 自动下载分析

### Changed
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
- refactor: 工作流步骤重编 + report §4 需求映射优化
- refactor: 清理死代码和 afprd 残留
- refactor: 删除冗余 schemas/ 和 contracts/ 目录
- refactor: 瘦身 v2.0 — 砍门禁 + 拆团队模式 + 去聚合化 + 恢复 portal

### Fixed
- fix: 修复 quality-gate.py 运行时崩溃 + 清理孤立文件

- 团队模式不再跳过 Step 2.5/3.5，改为从 snapshots 加载多仓 index，Step 3.1 增加 index_query 数据源

## [2.19.0] - 2026-05-13

- team common reference scaffolding + ingest-docx.py + self-audit 全盘修复

## [2.18.1] - 2026-05-12

- Evidence Index 准确性提升（多行签名、跨文件边、增量更新）

## [2.18.0] - 2026-05-12

- Artifact Contract MVP + Context Budget + Two-Pass Critic 自检

## [2.17.0] - 2026-05-12

- Workflow State 顺序验证 + report review checkpoint + stage 边界检测

## [2.16.3] - 2026-05-12

- 保真度优先架构修正 + 三段式工作流（spec/report/plan）+ prd-coverage-gate

## [2.16.1] - 2026-05-10

- 多层 benchmark + Evidence Index benchmark

## [2.16.0] - 2026-05-08

- 支持 .docx 输入 + Claude 多模态图片分析

## [2.15.0] - 2026-05-08

- 重写 README，强化对外可读性

## [2.14.0] - 2026-05-08

- 移除第三方依赖，精简产出结构

## [2.13.0] - 2026-05-08

- 移除 GitNexus/Graphify 依赖，回归原生能力

## [2.12.0] - 2026-05-08

- 状态面板 MVP + 安装流程统一

## [2.11.0] - 2026-05-07

- 输出目录统一为 _prd-tools/ + README + SKILL.md 流程图

## [2.9.0] - 2026-05-06

- 输出契约全面升级 + 契约校验自动化

## [2.8.0] - 2026-05-06

- 质量复盘：图片 confidence + questions 合并 + plan 技术方案升级

## [2.6.0] - 2026-05-04

- MarkItDown 集成 + LLM Vision + 多格式支持

## [2.5.1] - 2026-05-01

- 图谱增强字段补全（layer-impact + contract-delta）

## [2.5.0] - 2026-04-30

- 双维度影响分析（GitNexus 代码 + Graphify 业务）

## [2.4.1] - 2026-04-29

- 口径一致性修复 + 输出文件职责边界约束

## [2.4.0] - 2026-04-29

- 渐进式披露输出（report 9 章 + plan 可执行手册）

## [2.3.0] - 2026-04-29

- 兼容 v4.0 六文件 reference 结构

## [2.2.0] - 2026-04-29

- PRD 工程化解析（prd-ingest）+ 读取质量门禁

## [2.1.0] - 2026-04-28

- Layer Impact 改为能力面 surface + 输出瘦身

## [2.0.0] - 2026-04-28

- 完整 7 步蒸馏工作流 + 契约差异分析 + 反馈回流

## [1.1.0] - 2026-04-27

- 确定性事实校验 + 分级质量门控

## [1.0.0] - 2026-04-27

- 初始版本：3 步工作流 + Plugin manifest
