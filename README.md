# huntai-sre

个人 AI 应用项目，采用「文档先行、分阶段推进」的 Agent Team 协作流程：全部设计资产落盘在 `docs/`，实现代码只在 Stage 11 进入 `frontend/` 与 `backend/`。

> 当前阶段：**Stage 2 项目初始化与规范搭建**。仓库不含前后端实现代码。

## 立项信息（已核实 vs 未核实）

| 条目 | 状态 | 来源 |
|---|---|---|
| 仓库名 `huntai-sre` | 既定 | 工作区路径、`.gitleaks.toml` `title` |
| 产品名 HuntAI SRE | 既定（命名） | 同系列研究库 `opensource-product-analysis` 赛道文档（2026-09-13） |
| 产品形态：企业内部「可信 AI 运维层」 | **[ASSUMPTION]** | 研究库赛道结论，**不是**本仓库已批准 PRD。`decision/huntai/huntai-sre` 在研究库中仍为空。需仓库维护人在 Stage 3–5 / Stage 8 确认 |
| 不做全栈可观测平台 | **[ASSUMPTION]** | 同上；需 Stage 3 调研与 Stage 8 PRD 确认 |
| 内部前身 SuperBizAgent（FastAPI + LangGraph Demo） | 研究输入 | 研究库 `competitors/sre-agent-analysis/super-biz-agent/`；禁止当作本仓库已实现能力 |
| 目标客户栈、出域合规、IRM vs 纯调查切面 | **[TBD-BIZ]** | 仓库维护人于 Stage 3 调研与 Stage 8 PRD 确认 |
| 身份方案（OIDC / 其他） | **[TBD-BIZ]** + **[TBD-INFRA]** | 仓库维护人于 Stage 6 架构确认 |
| 运行时精确版本（Python / Node / PostgreSQL） | **已冻结** | [`docs/06_architecture_design/tech_stack.md`](docs/06_architecture_design/tech_stack.md)（ADR-0008）。Stage 11 写入锁文件 |

本仓库已生效的工程基线（Stage 1）：`.gitignore`、`.env.example`（仅键名）、pre-commit + gitleaks、LangChain 文档 MCP（`.mcp.json` 与 `.cursor/mcp.json`）。

## 目录结构

```text
huntai-sre/
├── AGENTS.md                   # 仓库级 Agent 指令（Cursor / ZCode）
├── CLAUDE.md                   # Claude Code 入口，指向 AGENTS.md
├── README.md                   # 本文件：总览与技术栈推荐
├── frontend/                   # 前端源码（Stage 11 落地，当前仅 README）
├── backend/                    # 后端源码（Stage 11 落地，当前仅 README）
└── docs/                       # 全部设计资产与过程文档的唯一存放地
    ├── README.md               # docs 索引与 0.8 文件映射
    ├── 00_setup/               # Stage 2  · 项目规范
    ├── 01_market_research/     # 业务调研
    ├── 02_competitor_analysis/ # 竞品分析
    ├── 03_problem_modeling/    # 业务问题建模
    ├── 04_interaction_design/  # 核心交互链路设计
    ├── 05_prototype/           # 产品原型规范
    ├── 06_architecture_design/ # 系统架构设计
    ├── 07_backend_design/      # 数据模型与 API 规范
    ├── 08_prd/                 # PRD
    ├── 09_figma_highfi/        # 高保真设计
    ├── 10_ai_context/          # AI 上下文
    ├── 11_test/                # 前端/后端实现与联调测试
    ├── 12_deployment/          # 发布和部署
    └── 13_changes/             # 变更记录（各阶段持续更新）
```

各目录的存放内容与约定输出文件见其内 `README.md`。阶段文件路径见根 `AGENTS.md` §3。

## 技术栈决策

选型原则：**单人项目没有团队缓冲去消化新栈的学习与踩坑成本**，因此选择「本人已在同系列仓库用过、生态与 AI 语料足够、能独立运维」的组合，而不是当前最流行的栈。

**已冻结**（2026-09-15，ADR-0008）：版本、License、维护活跃度、高危 CVE 审计见 [`docs/06_architecture_design/tech_stack.md`](docs/06_architecture_design/tech_stack.md)。Stage 11 才写入 `uv.lock` / `package-lock.json`。升 minor/major 须 change_log + 显式批准。

### 后端：Python 3.13 · FastAPI · uv · SQLAlchemy 2.0 · PostgreSQL 18

| 选择 | 理由 |
|---|---|
| Python 3.13 | 个人主力语言；LangChain / LangGraph 生态第一语言；与 uv 配套。3.12 已 security-only，不锁 3.14 新 feature 系列 |
| FastAPI + uvicorn | 原生 async；Pydantic v2 校验 + OpenAPI 作前后端契约；内部前身同栈 |
| uv | 虚拟环境与依赖一体化；`uv.lock` 可复现；已写入 `AGENTS.md` 与 `project_rules.md` |
| Pydantic Settings | 以仓库根 `.env` 为唯一配置入口 |
| SQLAlchemy 2.0（async）+ Alembic + psycopg 3 + PostgreSQL 18（≥18.6） | 审批 / 审计 / 配额 / 恢复禁止 SQLite（ADR-0005）。只保留一个驱动，供控制面与 LangGraph checkpointer 共用 |
| LangGraph（调查图，非业务控制面） | 有界只读工具循环。控制面仍在 FastAPI（ADR-0002）。禁止把模型当作权限 / 审批 / 重试 / 幂等的决策者 |

### 前端：React 19 · TypeScript 6 · Vite 8 · Tailwind CSS 4 · shadcn/ui

| 选择 | 理由 |
|---|---|
| React 19 + TypeScript 6 | 个人最熟练。不采用 TypeScript 7：无稳定 compiler API，eslint 不兼容 |
| Vite 8 纯 SPA | 前后端分离：渲染在客户端，服务端只有 FastAPI；静态资源由 FastAPI 同源托管 |
| Tailwind CSS 4 + shadcn/ui（源码内置） | 组件源码进仓库而非黑盒 npm 包，适合单人改调查时间线 / 审批卡 |
| TanStack Query + Zustand | 服务端状态（含调查进度轮询）与本地 UI 状态分开，不引入 Redux |

### 质量工具链（Stage 11 随代码初始化落地）

- 后端门禁：`ruff check` / `ruff format --check` / `mypy` / `pytest`
- 前端门禁：ESLint / `tsc --noEmit` / `npm run build` / Vitest
- 提交门禁：pre-commit + gitleaks（已配置）

一期测试框架即 pytest 与 Vitest + React Testing Library。**不**引入 Playwright。发布形态：Stage 12 `docker-compose.yml`（api + worker + postgres）；一期不上 Kubernetes。

## 关键文档

| 文档 | 内容 |
|---|---|
| `AGENTS.md` | 仓库级 Agent 指令：目录职责、`.env` / uv、阶段输出文件、Plan 优先 |
| `docs/00_setup/project_rules.md` | 命名、目录组织、Git / Conventional Commits、附录 D Checklist、测试验证、AI 红线 |
| `docs/README.md` | docs 分层索引与手册 0.8 文件映射 |
| `docs/13_changes/change_log.md` | 冻结资产与规范变更登记 |

个人级跨项目偏好：`~/.zcode/AGENTS.md`（不入库）。

## 阶段进度

| 阶段 | 产出位置 | 状态 |
|---|---|---|
| Stage 1 · 基础环境与安全基线 | 根目录忽略规则 / `.env.example` / pre-commit | ✅ 完成（本仓库已有提交） |
| Stage 2 · 初始化与规范搭建 | `docs/00_setup/` + 根 `AGENTS.md` | ✅ 本阶段完成 |
| 业务调研 | `docs/01_market_research/` | 待启动 |
| 竞品分析 | `docs/02_competitor_analysis/` | 待启动（外部研究包可引用，不可替代本目录约定输出） |
| 业务问题建模 | `docs/03_problem_modeling/` | 待启动 |
| 核心交互链路设计 | `docs/04_interaction_design/` | 待启动 |
| 产品原型规范 | `docs/05_prototype/` | 待启动 |
| 系统架构设计 | `docs/06_architecture_design/` | 进行中（Draft 架构 + **技术栈已冻结** ADR-0008） |
| 数据模型与 API 规范 | `docs/07_backend_design/` | 待启动 |
| PRD | `docs/08_prd/` | 待启动 |
| 高保真设计 | `docs/09_figma_highfi/` | 待启动 |
| AI 上下文 | `docs/10_ai_context/` | 待启动 |
| 实现与联调测试 | `docs/11_test/` + 代码目录 | 待启动 |
| 发布和部署 | `docs/12_deployment/` | 待启动 |
| 变更记录 | `docs/13_changes/` | ✅ 机制启用（`change_log.md`） |
| Stage 14 · 设计资产冻结与约束补充 | 根 `AGENTS.md` 补充条款 | 待设计冻结后启动 |
