# ADR-0001 模块化单体

- **Status**: Proposed（Draft 架构；非已批准产品冻结）
- **日期**: 2026-09-14
- **决策者**: 仓库维护人（待确认）

## 上下文

NFR-010 一期 20 注册账号，结构按 1000。单人运维。需要同库事务覆盖 Alert upsert、配额、审计、（二期）自动开查判定。

## 决策

一个对外 FastAPI 进程（控制面 HTTP）+ 一个 worker 进程（调查图与扫描）。逻辑模块分界见 `01_domain_and_service_architecture.md`。不拆微服务。

## 替代

- **多微服务**（ingest / investigation / scan 分仓）：隔离更好，运维与分布式事务成本与 20 人规模不匹配。
- **纯单进程内线程**：崩溃会同时丢掉 API 与长任务；弱于「API + worker」。

## 后果

- 一期部署简单；多实例时用 PostgreSQL 锁保证 auto 并发与配额（`03` `[TBD-INFRA]`）。
- Stage 12 若要拆分，模块边界已按领域切开，不必先上总线。
