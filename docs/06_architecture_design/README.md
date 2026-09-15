# 06_architecture_design — 系统架构设计

存放系统架构设计产出：运行时拓扑、模块边界、前后端职责、前端页面与技术栈冻结。
由系统架构设计阶段产出。

**约定输出**：

- `architecture_spec.md` — 系统架构（Markdown）
- `frontend_design_spec-v1.0.md` — 前端设计规范（Markdown）
- `frontend_backend_boundary_spec-v1.0.md` — 前后端边界（Markdown）

手册 0.8 将 `architecture_spec.md` 列在 `07_backend_design/`；本仓库按目录职责放在本目录。

## 本阶段 Draft 索引（2026-09-14 后端架构 Agent Team）

全部 **Status: Draft**（系统架构），非已批准产品结论。例外：[`tech_stack.md`](./tech_stack.md) / ADR-0008 **已冻结**。`frontend_design_spec-v1.0.md` **先不写**（维护人 2026-09-14：等 Stage 7 API 契约后再做前端架构），故 Stage 6 约定输出尚未齐。

| 文件 | 说明 |
|---|---|
| [`00_research_and_input_traceability.md`](./00_research_and_input_traceability.md) | 输入追溯与官方调研 |
| [`01_domain_and_service_architecture.md`](./01_domain_and_service_architecture.md) | 领域与服务 |
| [`02_api_workflow_and_review.md`](./02_api_workflow_and_review.md) | 接口、任务、审核 |
| [`03_security_reliability_and_operations.md`](./03_security_reliability_and_operations.md) | 安全与运行 |
| [`04_architecture_review.md`](./04_architecture_review.md) | 审查 |
| [`architecture.md`](./architecture.md) | 集成入口（与 spec 同文） |
| [`architecture_spec.md`](./architecture_spec.md) | Stage 6 约定集成正文 |
| [`frontend_backend_boundary_spec-v1.0.md`](./frontend_backend_boundary_spec-v1.0.md) | 前后端边界原则（非 OpenAPI） |
| [`tech_stack.md`](./tech_stack.md) | **已冻结**：实现技术栈、版本、License、CVE 审计 |
| [`adr/`](./adr/) | ADR-0001–0007 Proposed；**ADR-0008 Accepted**（技术栈） |
