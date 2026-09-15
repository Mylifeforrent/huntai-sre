# HuntAI SRE 技术栈冻结

- **Status**: 已冻结（仅技术栈；系统架构其余部分仍为 Draft）
- **日期**: 2026-09-15
- **审计日期**: 2026-09-15
- **ADR**: [`adr/0008_tech_stack.md`](./adr/0008_tech_stack.md)
- **输入**: [`architecture_spec.md`](./architecture_spec.md)、[`frontend_backend_boundary_spec-v1.0.md`](./frontend_backend_boundary_spec-v1.0.md)、ADR-0001/0002/0005/0007、根 README 推荐组合、`project_rules.md` §5
- **非目标**: 不生成实现代码；不写 `docker-compose.yml` / `.github/`（Stage 12）；不预装无 BR 的库

审计来源：OSV `querybatch`（所列精确版本 0 条漏洞）；已知历史高危 CVE 核对 GitHub Advisory / PostgreSQL 安全公告，确认当前锁定版本已含补丁。

**落锁规则**：Stage 11 写入 `uv.lock` / `package-lock.json` 时以本表版本为基线。允许同 minor 的更新 patch（须复跑 OSV，高危/严重 = 0）。升 minor / major 必须 change_log + 用户批准。

---

## 1. 选型原则与拒绝清单

原则：单人能独立运维；与前身 SuperBizAgent（FastAPI + LangGraph）和 huntai 系列约定同向；只加实现一期北极星（告警 → 调查 → cited RCA / inconclusive）所必需的技术。

| 不采用 | 理由 |
|---|---|
| Next.js / SSR | 渲染在浏览器，服务端只有 FastAPI（边界规范 §1） |
| Redux / MobX | 服务端状态交给 Query，本地 UI 用 Zustand |
| GraphQL / gRPC | 单一 Web 客户端；路径风格已由 `project_rules` 定为 REST |
| SSE 作为一期进度权威 | ADR-0007：GET 轮询权威；SSE 仍可选、不进本期依赖 |
| TypeScript 7 | 无稳定 compiler API；`typescript-eslint` 不兼容（跟踪到 7.1） |
| asyncpg | 与 LangGraph checkpointer 的 psycopg 重复 |
| Celery / Redis / Kafka / RabbitMQ | 进程内领域事件 + PostgreSQL；20 人规模 |
| Playwright | 产品永不做（Business Model §9）；E2E 留 Stage 12 再议 |
| Kubernetes 编排 | 一期单实例；Stage 12 用 compose，不宣称高可用 |
| 运行时 MCP / LangGraph HITL / langgraph-api | ADR-0003 / 0004；只嵌库，不上 Host |
| Sentry SDK | `.env` 未配 DSN 则不上报；不预装 |
| 具体 LLM 厂商 SDK | BYOK / 网关产品名 `[TBD-BIZ]`；Stage 11 再 `uv add` 对应适配器 |

---

## 2. 决策摘要（实现必需）

| 项 | 冻结选择 | 选择理由 |
|---|---|---|
| 前端框架 | React 19 + TypeScript 6 + Vite 8 纯 SPA；路由 `react-router` 8 | 个人最熟练；AI 语料充足；与 FastAPI 职责不重叠 |
| UI 组件 | Tailwind CSS 4 + shadcn/ui（CLI 复制源码进仓库） | 调查时间线 / 审批卡需改组件源码，不要黑盒 npm 包 |
| 状态管理 | TanStack Query 管服务端缓存与轮询；Zustand 管页内 UI | 进度 GET 轮询是 Query 的本职；不引入 Redux |
| 后端框架 | CPython 3.13 + FastAPI + uvicorn；包管理 uv | 原生 async；Pydantic v2 + OpenAPI；前身同栈；uv 已写入工程规范 |
| ORM / 迁移 | SQLAlchemy 2.0 async + Alembic；驱动 **仅** psycopg 3 | ADR-0005；一个驱动同时服务控制面与 LangGraph checkpointer |
| API 风格 | REST JSON，`/api/v1/` + 小写连字符复数名词；写命令 `Idempotency-Key`；进度 GET 权威 | 已写入 `project_rules` 与 `02` §1；具体 path Stage 7 冻结 |
| 调查运行时 | LangGraph **库**内嵌 worker；PostgreSQL checkpointer | ADR-0002；禁止 InMemorySaver 上生产 |
| 测试框架 | 后端 pytest + pytest-asyncio + httpx；前端 Vitest + React Testing Library | 与 `project_rules` §5 合入命令一致 |
| 构建部署 | 开发：Vite（禁止 `--host`）+ uvicorn + worker + 本机 PostgreSQL。发布：Vite `build` 静态资源由 FastAPI 同源托管。Stage 12：compose 三容器（api / worker / postgres）。一期不上 K8s、不上独立 Nginx 集群 | 单人运维；NFR 不写 SLA |

---

## 3. 运行时（非 PyPI/npm 包）

| 组件 | 锁定 | License | 维护活跃度 | 高危/严重 CVE（2026-09-15） |
|---|---|---|---|---|
| CPython | **3.13.x**（基线 3.13.15，2026-08-05） | PSF | bugfix 至 2029-10；不锁 3.12（已 security-only）、不锁 3.14（新 feature 系列） | 采用当月 latest 3.13 补丁；实现前再核 python.org 安全页 |
| Node.js | **24.x Active LTS**（基线 24.21.0） | MIT | Active LTS 至 2026-10-20，安全至 2028-04-30。不用 22（维护期）、不用 26 Current | 采用当时 24.x latest patch |
| PostgreSQL | **18.x，最低 18.6** | PostgreSQL License | 当前稳定主线，支持至 2030-11 | 18.6（2026-08-13）修复 CVE-2026-14664 等一批 8.8 级问题；**禁止** 18.0–18.4。18.5 未发行 |

---

## 4. 后端核心依赖

| 包 | 锁定版本 | License | 维护活跃度 | 高危/严重 CVE |
|---|---|---|---|---|
| fastapi | 0.141.1 | MIT | 2026-07-29 发版；官方持续维护 | OSV 0。Starlette 历史高危（multipart DoS、Windows StaticFiles SSRF）已在 starlette ≥1.1 / 本线 1.6.0 修复 |
| starlette | 1.6.0（随 FastAPI 解析，禁止降到 1.0.x） | BSD-3-Clause | 2026 仍发 1.x | OSV 对 1.6.0 为 0 |
| uvicorn | 0.53.0 | BSD-3-Clause | 活跃 | OSV 0 |
| pydantic | 2.13.5 | MIT | 活跃（v2） | OSV 0 |
| pydantic-settings | 2.15.0 | MIT | 活跃；读仓库根 `.env` | OSV 0 |
| SQLAlchemy | 2.0.53 | MIT | 2.0 稳定线 2026-08 仍发版；不采用 2.1 rc | OSV 0 |
| alembic | 1.20.0 | MIT | SQLAlchemy 同项目组 | OSV 0 |
| psycopg | 3.3.5（`psycopg[binary]`） | **LGPL-3.0-only** | 活跃。内部工具动态链接可接受；禁止再引入 asyncpg | OSV 0 |
| httpx | 0.28.1 | BSD-3-Clause | 活跃；Prom / Loki / LLM 网关出站 | OSV 0 |
| langgraph | 1.2.11 | MIT | 2026 持续发 1.2.x | OSV 0。须 ≥1.0.10（CVE-2026-28277 不安全 checkpoint 反序列化） |
| langgraph-checkpoint | 4.2.0 | MIT | 与上同仓 | OSV 0。须 ≥4.0.0（CVE-2026-27794 pickle fallback） |
| langgraph-checkpoint-postgres | 3.1.2 | MIT | 与上同仓 | OSV 0。须 ≥3.1.1（CVE-2026-71433 命名空间前缀越界，中危，仍要求打上） |
| langchain-core | 1.6.3 | MIT | 活跃；图与消息类型 | OSV 0 |
| authlib | 1.8.0 | BSD-3-Clause | 活跃；OIDC 客户端。IdP 产品名仍 `[TBD-BIZ]` | OSV 0 |
| kubernetes | 36.0.3 | Apache-2.0 | 官方客户端，跟 K8s API。只读工具 + 二期写身份 | OSV 0 |
| uv | 0.12.13（开发/CI 工具链） | MIT OR Apache-2.0 | 活跃 | OSV 0 |

质量门禁（`project_rules` §5，Stage 11 落地，非运行时）：

| 包 | 锁定版本 | License | 活跃度 | 高危/严重 CVE |
|---|---|---|---|---|
| pytest | 9.1.1 | MIT | 活跃 | OSV 0 |
| pytest-asyncio | 1.4.0 | Apache-2.0 | 活跃 | OSV 0 |
| ruff | 0.16.7 | MIT | 活跃 | OSV 0 |
| mypy | 2.3.1 | MIT | 活跃 | OSV 0 |

---

## 5. 前端核心依赖

| 包 | 锁定版本 | License | 维护活跃度 | 高危/严重 CVE |
|---|---|---|---|---|
| react / react-dom | 19.3.0 | MIT | 2026-09-09 发版 | OSV 0 |
| vite | 8.3.0 | MIT | 2026-09-10 发版 | OSV 0。8.0.0–8.0.4 / ≤8.0.15 曾有开发服务器任意文件读取（CVE-2026-39363 等，高危）；**必须 ≥8.0.16，本冻结 8.3.0**。生产静态资源不受影响。开发禁止 `--host` |
| @vitejs/plugin-react | 6.1.1 | MIT | 与 Vite 8 匹配 | OSV 0 |
| typescript | **6.0.3** | Apache-2.0 | 6.0 为仍带 compiler API 的稳定线 | OSV 0。不采用 7.0.2：无稳定 API，eslint 崩溃 |
| react-router | 8.3.1 | MIT | 2026-08-28；SPA 多页 PG-001–012 必需 | OSV 0 |
| tailwindcss | 4.3.3 | MIT | 2026-07 发 4.3 | OSV 0 |
| shadcn（CLI） | 4.21.0 | MIT | 活跃。组件复制进 `frontend/src/components/ui`，入库后按仓库 diff 维护 | OSV 0 |
| @tanstack/react-query | 5.102.8 | MIT | 2026-08-27 | OSV 0 |
| zustand | 5.0.15 | MIT | 2026-08-13 | OSV 0 |

shadcn 复制时会带入少量样式工具（`clsx`、`tailwind-merge`、`class-variance-authority`、图标 `lucide-react`）。它们不是独立选型，Stage 11 随 CLI 生成；合入前按 R4 审计锁文件，不在本文预扩依赖清单。

质量门禁：

| 包 | 锁定版本 | License | 活跃度 | 高危/严重 CVE |
|---|---|---|---|---|
| vitest | 5.0.0 | MIT | 2026-09-03；对齐 Vite 8 | OSV 0 |
| @testing-library/react | 16.3.3 | MIT | 2026-08-27 | OSV 0 |
| jsdom | 30.0.1 | MIT | Vitest 环境 | OSV 0 |
| eslint | 10.10.0 | MIT | 2026-09-04 | OSV 0 |
| typescript-eslint | 8.70.0 | MIT | 活跃；peer 要求 TypeScript &lt; 7 | OSV 0 |

---

## 6. API、测试、构建部署（行为冻结）

### 6.1 API 风格

- **必须** REST + JSON；前缀 `/api/v1/`；资源名词复数、小写连字符。
- 创建调查等命令同步返回资源 id；调查运行异步；**进度以 GET 为权威**（ADR-0007）。
- 可能重复提交的写命令接受 `Idempotency-Key`（调查创建、扫描、二期 confirm）。
- 错误 envelope 与 HTTP/业务码映射见 `02` §1.3；无 GraphQL schema、无 gRPC。
- OpenAPI 由 FastAPI 生成，Stage 7 写字段契约，Stage 11 与实现对照。

### 6.2 测试

- 后端：`backend/tests/`；`uv run ruff check . && uv run ruff format --check . && uv run mypy app && uv run pytest`。
- 前端：`npm run lint && npx tsc --noEmit && npm run build && npm run test`。
- 一期 **不** 引入 Playwright。产品永不把浏览器自动化当调查工具。

### 6.3 构建与部署

| 阶段 | 方式 |
|---|---|
| 本地开发 | 根 `.env`；PostgreSQL 本机或单容器；`uvicorn` 控制面；第二进程跑 worker；Vite 只绑定 localhost，**禁止** `--host` |
| 前端交付物 | `vite build` → 静态文件；由 FastAPI 同源托管。生产不单独开前端源站，避免 CORS `*` |
| Stage 12 编排 | 顶层 `docker-compose.yml`：**api、worker、postgres** 三服务。不在本期创建该文件 |
| 一期明确不做 | Kubernetes、多副本、外部任务队列、独立对象存储（证据默认库内截断，对象存储仍 `[TBD-INFRA]`） |
| CI | Stage 12 才允许 `.github/`；门禁命令必须与 `project_rules` §5 一致 |

---

## 7. 审计自检

- [x] 上表每个锁定版本已对 OSV 查询；**高危/严重 CVE = 0 未处置**。LangGraph / Vite / PostgreSQL / Starlette 的历史高危均落在低于本冻结下限的版本，已用版本下限消化。
- [x] 每个选型为仓库维护人可独立运维：无微服务、无 K8s、无第二套状态库、无 SSR 框架。
- [x] 未为「看起来完整」加入 Redis、GraphQL、Playwright、Sentry SDK、LLM 厂商包。
- [x] 未在 Stage 11 之前生成 `.py` / `.ts` / `.tsx` 实现文件。

复扫命令（Stage 11 落锁后必须再跑，不能只信本文快照）：

```bash
# 后端（锁文件存在后）
uv run pip-audit --strict
# 或：osv-scanner --lockfile backend/uv.lock

# 前端
cd frontend && npm audit --audit-level=high
# 或：osv-scanner --lockfile frontend/package-lock.json
```

本文审计因仓库尚无锁文件，对 **将要锁定的精确版本** 使用 OSV API 直查，而不是 `npm audit`（无 `package-lock.json`）。
