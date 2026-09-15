# ADR-0008 一期实现技术栈冻结

- **Status**: Accepted
- **日期**: 2026-09-15
- **决策者**: 仓库维护人（当次任务：确定并冻结技术栈）
- **权威正文**: [`../tech_stack.md`](../tech_stack.md)

## 上下文

根 README 把 FastAPI / React / Vite / PostgreSQL 写成工作假设。ADR-0001/0002/0005 已把运行时形态定为「FastAPI 控制面 + worker + PostgreSQL + LangGraph 仅调查图」，但未锁版本。Stage 11 禁止在未冻结栈上生成实现代码。个人项目没有团队缓冲去消化新栈。

## 决策

一期只冻结实现所必需的技术，版本、License、活跃度与高危 CVE 见 `tech_stack.md`。摘要：

| 项 | 选择 |
|---|---|
| 前端框架 | React 19 + TypeScript 6 + Vite 8 SPA |
| UI | Tailwind CSS 4 + shadcn/ui 源码内置 |
| 状态 | TanStack Query（服务端）+ Zustand（本地 UI） |
| 后端 | CPython 3.13 + FastAPI + uvicorn + uv |
| ORM | SQLAlchemy 2.0 async + Alembic；驱动仅 psycopg 3 |
| API | REST JSON `/api/v1/`；进度 GET 轮询（ADR-0007） |
| 调查图 | LangGraph 库内嵌于 worker；checkpointer 用 PostgreSQL |
| 测试 | 后端 pytest；前端 Vitest + React Testing Library |
| 构建部署 | Vite 静态构建由 FastAPI 同源托管；Stage 12 才写 compose（api + worker + postgres）。一期不上 Kubernetes、不上外部队列 |

## 替代（拒绝）

- Next.js / SSR：与「服务端只有 FastAPI」重叠。
- Redux / GraphQL / gRPC：无多端消费者，增加单人运维面。
- TypeScript 7：无稳定 compiler API，`typescript-eslint` 不兼容。
- asyncpg + psycopg 双驱动：LangGraph Postgres checkpointer 已用 psycopg，只保留一个驱动。
- Celery / Redis / Kafka：NFR-010 一期 20 人；进程内领域事件 + PostgreSQL 锁足够。
- Playwright / Kubernetes：产品永不做 Playwright（Business Model §9）；编排留给 Stage 12，且一期不宣称高可用。

## 后果

- Stage 11 `uv add` / `npm install` 必须以 `tech_stack.md` 冻结基线为准；升 major / minor 须 change_log + 显式批准。
- 同 minor 的安全 patch 允许在落锁时上调，前提是 OSV 高危/严重仍为 0。
- LLM 厂商 SDK、IdP 产品名仍 `[TBD-INFRA]` / `[TBD-BIZ]`，不预装。
