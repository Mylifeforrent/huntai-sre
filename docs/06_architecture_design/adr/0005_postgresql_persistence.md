# ADR-0005 PostgreSQL 作为控制面与审计存储

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

`.env.example` 已预留 `POSTGRES_PASSWORD`。README 假设禁止用 SQLite 承担审批 / 审计 / 恢复。需要：Alert 幂等 upsert、配额原子计数、auto 并发计数、append-only 审计、可选 LangGraph 耐久 checkpointer。

## 决策

控制面、审计、配额、ApprovalRequest 使用 **PostgreSQL**（SQLAlchemy 2 async + Alembic 在 Stage 11 落地）。SQLite 不用于上述职责。大 ToolCall 原文默认截断后入库；对象存储 `[TBD-INFRA]`。

## 替代

- SQLite：无法安全支撑并发 upsert 与备份意图。
- 先上独立事件总线 + 多库：无 NFR 要求，拒绝。

## 后果

Stage 11 初始化依赖用户批准后 `uv add`。运行时 PostgreSQL **18.x 最低 18.6**、SQLAlchemy 2.0.53、Alembic 1.20.0、驱动仅 psycopg 3.3.5，见 ADR-0008 / `tech_stack.md`。对象存储仍 `[TBD-INFRA]`。
