# HuntAI SRE 系统架构规范

- **Status**: Draft（Stage 6 集成草案；**非已批准、非已冻结**）
- **日期**: 2026-09-14
- **文件名**: 本文是当次 Agent Team 要求的集成入口 `architecture.md`。Stage 6 约定名 [`architecture_spec.md`](./architecture_spec.md) 与本文 **正文相同**。
- **输入**: [`../03_problem_modeling/business_model.md`](../03_problem_modeling/business_model.md)（优先）；[`../04_interaction_design/interaction_design_summary.md`](../04_interaction_design/interaction_design_summary.md)；追溯 [`00_research_and_input_traceability.md`](./00_research_and_input_traceability.md)
- **原型**: `05_prototype` 无链接/截图，不阻塞、不补造。
- **非目标**: 实现代码、迁移、编排、Stage 7 OpenAPI、前端视觉规范。

专题正文：[`01_domain_and_service_architecture.md`](./01_domain_and_service_architecture.md) · [`02_api_workflow_and_review.md`](./02_api_workflow_and_review.md) · [`03_security_reliability_and_operations.md`](./03_security_reliability_and_operations.md) · [`04_architecture_review.md`](./04_architecture_review.md) · 边界 [`frontend_backend_boundary_spec-v1.0.md`](./frontend_backend_boundary_spec-v1.0.md)

---

## 1. 架构目标、范围与非目标

**目标**: 让被 xMatters 叫起的人，用深链或列表打开同一条 Alertmanager firing 告警，在 HuntAI 内完成可区分的 cited RCA 或 `inconclusive`（G1）。无模型时仍能看 firing 告警，SRE 能跑零 LLM 扫描（G3）。

**范围**

- 一期：OIDC + break-glass、TeamBinding、2–5 ClusterSource、AM webhook 入库、人工调查、只读工具、服务端 Citation、成本闸、Skill、RelatedLink、按需扫描、审计与保留、Web API。
- 二期边界（不进一期验收）：critical 自动开查、Loki LogQL citation、人批 restart/scale、60 分钟扫描与开发只读 Finding。

**非目标**: 全栈可观测、IRM、替换 AM/xMatters、silence/ack/page/IM/邮件、kubectl delete/apply/exec、读 secrets、Playwright、知识库上传、无 `alert_id` 对话、对外 SaaS、MCP 运行时工具总线、把 LangGraph 当控制面。完整清单见 Business Model §9。

---

## 2. 输入追溯与关键假设

完整表见 `00`。Lead 采用的关键假设：

| ID | 内容 | 若被推翻 |
|---|---|---|
| A02 | 无取消调查；能见 Alert 则可只读其调查 | 需改读 ACL 与 API |
| A03 | 同集群 running 扫描互斥 | 放开并行 ScanRun |
| A06 | 150s 只计 LLM+工具；pending 不耗墙钟、调查保持 running | 二期人批窗口失效或须改 BR（走 change_log） |
| 栈 | FastAPI + worker + PostgreSQL + 调查图用 LangGraph | ADR-0001/0002/0005 |

冲突处置：C1/C2 已按用户「按推荐执行」落盘；C3 用 A06 解释墙钟，不改 BR 正文；C4 `ALERT_WEBHOOK_URL` 不接入产品。

---

## 3. 系统上下文与模块职责

```text
Alertmanager ──webhook+IngestCredential──► Ingest ──upsert──► Alert
IdP OIDC / break-glass ──────────────────► API 控制面
xMatters ──仅运营深链 URL──► 浏览器 ──► API
控制面 ──只读 SA / Prom──► 集群
控制面 ──（二期）Loki / WriteIdentity──► 集群
调查 worker ──LangGraph 只读循环──► 控制面落 ToolCall / Claim 草案
控制面 ──脱敏后──► LLM 网关（可关）
```

模块化单体（ADR-0001）。模块表见 `01` §2。一句话：

- **控制面**拥有身份、告警、配额、扫描、配置、审计、Citation id、Investigation 主状态、二期审批与写执行。
- **调查运行时**只跑有界只读图（ADR-0002）。
- **证据库**存截断 ToolCall，不是用户上传。

---

## 4. 核心领域模型、状态机和数据流

实体以 Business Model §3 为闭集。Lead 不新增实体。

**主数据流（一期北极星）**

1. AM webhook → 鉴权 → fingerprint upsert Alert（不建调查）。
2. 人经 OIDC 打开深链/列表 → 可见性检查 → 「看过」审计。
3. 人确认「调查」→ 配额/闸/并发 → Investigation `running` → worker 拉 Skill → 只读工具 → 脱敏 → LLM → 控制面铸造 Citation → `completed` 或 `inconclusive`。
4. SRE 可另发零 LLM ScanRun。

**二期增量数据流**: firing + `severity=critical` → 条件创建 `trigger=auto`（≤1/firing，auto running≤5，组织闸阻断则跳过）→ 同一调查图。图可 `ProposeWriteAction` → 人 confirm → WriteIdentity。定时扫描独立于调查。

状态机与业务文档一致。有 pending 审批时延迟终态，主状态仍为 `running`（BR-090）。PG-006 L1：「只读调查已结束，等待确认写操作」。

---

## 5. API、异步任务、事件和前后端协作

- 风格：`/api/v1/` 复数连字符；具体 path Stage 7 冻结。深链参数 = `cluster_id` + fingerprint。
- 创建调查同步返回 id；运行异步；进度 **GET 权威**，SSE 可选（ADR-0007）。
- 错误不泄密；403 不泄露不可见告警正文。
- 写命令使用幂等键 `[ASSUMPTION]`。
- 进程内领域事件即可，不上外部总线。
- 无产品推送通知。

页态映射见 `02` §4。前端隐藏 ≠ 授权。

完整原则：[`frontend_backend_boundary_spec-v1.0.md`](./frontend_backend_boundary_spec-v1.0.md)。

---

## 6. 审核与人工介入

一期：**无** RCA 人工改 Claim，**无**写修复。人工介入 = 点调查、配数据源/绑定/闸、确认扫描。

二期：唯一审核对象是集群写修复（restart/scale）。preview → 第二次 confirm；模型不可自批；`auto-investigator` 不可 confirm。不设审批中心页。

**MCP**: 不采用运行时（ADR-0003）。  
**LangGraph HITL**: 不采用为账本/写执行（ADR-0004）。LangGraph 仅调查图。

---

## 7. 数据、安全、可靠性、可观测性与运维

- 认证：OIDC + 唯一 break-glass；ingest 分离凭证。
- 授权：单组织；标签 TeamBinding；服务端行级过滤。
- 数据：PostgreSQL（ADR-0005）；保留 90/365 天；无清单不出域。
- 可靠性：有界闸 + 幂等 webhook + worker 崩溃不得标 completed；不宣称高可用。
- 可观测：结构化日志（禁密钥与未脱敏原文）；Sentry 可选且须脱敏；产品不用 `ALERT_WEBHOOK_URL`。
- 运维：根 `.env`；Stage 12 再写编排；多副本锁 `[TBD-INFRA]`。

---

## 8. 技术调研结论（MCP / HITL）

| 问题 | 结论 | 理由摘要 |
|---|---|---|
| 运行时 MCP | **不采用** | 官方 Host/Client/Server 与用户同意模型不适合写死白名单；无 BR；文档 MCP 仅编码助手 |
| LangGraph | **采用（调查图）** | 有界工具循环；控制面仍在 FastAPI |
| LangGraph HITL | **不采用（审批/写）** | interrupt 无限等待 + 节点重跑 vs BR-082/085/040 与 WriteIdentity |

调研条目与链接见 `00` §3。

---

## 9. ADR 摘要

| ADR | 推荐 | 替代 |
|---|---|---|
| [0001](./adr/0001_modular_monolith.md) | FastAPI + worker 单体 | 微服务（过重） |
| [0002](./adr/0002_control_plane_vs_investigation_runtime.md) | 控制面 / 调查图分离 | 全进图（拒绝） |
| [0003](./adr/0003_no_runtime_mcp.md) | 无运行时 MCP | 每集群 MCP Server |
| [0004](./adr/0004_approval_request_not_langgraph_hitl.md) | ApprovalRequest + 控制面执行 | interrupt 当账本 |
| [0005](./adr/0005_postgresql_persistence.md) | PostgreSQL | SQLite |
| [0006](./adr/0006_phase1_slice_phase2_boundaries.md) | 一期关二期能力 | 只设计一期 |
| [0007](./adr/0007_investigation_progress_polling.md) | GET 轮询 | 仅 SSE |

全部 Proposed，待维护人接受后才可当冻结。

---

## 10. 风险

| 风险 | 级别 | 缓解 |
|---|---|---|
| A06 / BR-085 与人批窗口 | 中 | 已写入 BR-040 / BR-090；超时 `[TBD-BIZ]` |
| 无脱敏清单仍出域 | 高 | NFR-025 fail-close |
| 工具白名单被模型绕过 | 高 | 静态注册表，不 MCP 发现 |
| 人点并发/组织闸 TBD 导致实现猜数字 | 中 | 可插策略，禁止假默认 |
| 单实例无 SLA | 中 | 不对外承诺；Stage 12 再议 |
| 原型缺失导致页细节偏差 | 低 | 以交互摘要为准 |
| Stage 6 缺 `frontend_design_spec-v1.0.md` | 中 | **已拍板延后**：Stage 7 API 契约之后再写；不阻塞后端 Draft |

---

## 11. 开放问题（需用户确认）

已于 2026-09-14 收口：无 LLM 只跑工具（BR-053）；人点并发不硬拒绝（BR-054）；拒绝不占日额（BR-041）；组织闸阻断 auto（BR-042）；能见 Alert 可读他人调查（BR-055）；人批等待须补 L1 文案（BR-090 / IX-FB-11）；**前端设计规范等 Stage 7 API 契约后再写**。

仍开放：

1. IdP 产品名、cookie/bearer。
2. 临时放行额度与时限。
3. 组织日 token 默认数字；pending 人批等待超时秒数。
4. 脱敏清单、截断字节、出域网关地址、Skill Git、`scale_max`。
5. 开发深链探测（404 vs 403）；开发空绑定是否允许登录。
6. 证据对象存储 vs 仅库内；webhook 认证算法；多 worker 队列。
7. SSE 是否进入一期。

---

## 12. 验证计划（文档级，非实现）

- [ ] 每条模块职责能追溯到 BR 或明确 `[ASSUMPTION]`。
- [ ] 状态枚举 ⊆ Business Model §5（除 A06 延迟终态说明）。
- [ ] 一期 API 语义不包含自动开查/LogQL/写修复入口。
- [ ] MCP 运行时与 LangGraph HITL 账本均为「不采用」且有官方链接。
- [ ] 无 `.py/.ts` 新增；无密钥；无「高可用」空话。
- [ ] Agent D 审查文件已覆盖幂等、并发、失败恢复、审计、泄露。
- [ ] 用户接受 ADR 后：改 Status 须 change_log + 显式批准。

下一步合法阶段：Stage 7 数据模型与 API 契约；**不是** Stage 11 编码。
