# 变更登记表

任何对冻结资产或已沉淀阶段产出的修改，必须先在本表追加一行，再改文件。冻结定义见根 `AGENTS.md` §5。

| 日期 | 对象 | 原因 | 获批 | 执行结果 |
|---|---|---|---|---|
| 2026-09-13 | Stage 2 初始化：`AGENTS.md`、`CLAUDE.md`、`docs/` 分层、`project_rules.md`、根 `README.md` | 项目规范落盘 | 是（本阶段任务） | 已创建；无前后端实现代码 |
| 2026-09-14 | `docs/03_problem_modeling/business_model.md` | 增补二期范围（Loki 只读、人批 restart/scale、定时扫描、`severity=critical` 自动开查）。一期 MVP 20 项不改 | 是（仓库维护人 Q1–Q15 收口） | 已写入 §11、§2.5–2.7、BR-070–BR-089；§9 改为永不做 40 项 |
| 2026-09-14 | `docs/04_interaction_design/interaction_design_summary.md` | Business Model 二期增补后重放交互：自动开查、LogQL citation、人批写修复、定时扫描 Finding 只读、ClusterSource `env`/`write_remediation`/`scale_max`。一期页面与 TF-01–TF-12 不进入二期验收 | 是（用户要求按原规则重生成） | 已重写摘要：一期 PG-001–PG-012 / TF-01–TF-12 保持；新增〔二期〕TF-13–TF-17 与导航/人批/Finding 只读增量 |
| 2026-09-14 | `docs/03_problem_modeling/business_model.md`；`docs/04_interaction_design/interaction_design_summary.md`；`docs/06_architecture_design/` Draft | 收口架构开放问题：无 LLM 只跑工具；人点并发不硬拒绝；拒绝不占日额；组织闸阻断 auto；能见 Alert 则可读他人调查；人批等待文案；前端规范推迟到 Stage 7 之后 | 是（维护人逐条回复 1–7，2026-09-14） | 见各文件；未写 `frontend_design_spec-v1.0.md`；未填脱敏清单/截断字节/pending 超时等假默认数字 |
