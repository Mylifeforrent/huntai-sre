# 调研日志与输入追溯

- **Status**: Draft（Stage 6 后端架构 Agent Team；非已批准、非已冻结）
- **日期**: 2026-09-14
- **范围**: 只读输入整理 + 围绕待决策问题的官方一手调研。禁止把本文当实现说明书。
- **批准的计划假设**（用户 2026-09-14「按推荐执行」）:
  1. 专题文档 snake_case；集成正文同时满足 `architecture.md` 与 `architecture_spec.md`；本次不写 `frontend_design_spec-v1.0.md`；写 Draft 前后端边界原则。
  2. BR-040 墙钟 150s **只计 LLM + 工具执行**；`ApprovalRequest=pending` 期间 Investigation 保持 `running`、不消耗墙钟；超时秒数 `[TBD-BIZ]`。不改 `business_model.md`。
  3. 架构覆盖一期可部署切片 + 二期模块边界；一期验收仍只对 §8 的 20 项。

---

## 1. 实际读取的输入

| 路径 | 定位 | 本阶段用法 |
|---|---|---|
| `AGENTS.md`、`docs/00_setup/project_rules.md` | 目录职责、命名、红线、Stage 6 约定文件 | 工程约束；与当次指令的文件名冲突见 §4 C1 |
| 根 `README.md` | 推荐栈（现已由 ADR-0008 冻结） | 历史输入；现行版本以 `tech_stack.md` 为准 |
| `.env.example` | 已预留键 | 配置边界；`ALERT_WEBHOOK_URL` 见 C4 |
| `.mcp.json`、`.cursor/mcp.json` | 编码助手用 LangChain **文档** MCP | 不是产品运行时 |
| `docs/13_changes/change_log.md` | 二期增补已获批 | 二期边界合法进入架构，不进一期验收 |
| `docs/01_market_research/market_research_report.md` | 用户/场景/不做 MCP 全量 CRUD | 背景；不得覆盖 BR |
| `docs/02_competitor_analysis/competitor_analysis_report.md` | 可借鉴：cited、人批、成本闸、AI-Free；应避免：无审批自动修、allowlist 失效 | 取舍理由 |
| **`docs/03_problem_modeling/business_model.md`** | **后端核心事实** | 实体、BR、NFR、状态机的唯一业务权威 |
| **`docs/04_interaction_design/interaction_design_summary.md`** | 页面、TF/EX、页态、权限表现 | 接口与状态映射 |
| `docs/05_prototype/README.md` | 目录占位 | 无 `prototype_link.md` / 截图；**不阻塞、不补造** |
| `docs/06_architecture_design/README.md` | 约定输出文件名 | 起草前无 `architecture.md` |
| `docs/07_backend_design/README.md` 及后续 README | 未开始 | 本阶段不写表字段级契约 |

未读取仓库外研究包正文。`docs/` 下无架构相关图片。

---

## 2. 输入追溯

### 2.1 已确认事实（架构必须遵守）

| ID | 事实 | 来源 | 架构影响 |
|---|---|---|---|
| F01 | 公司内单 Organization；Web 唯一入口；不替代 Prom / AM / xMatters | BR-001–005、G4、MVP-20 | 单租户模块化单体；无多组织路由；无对 AM/xMatters 的写适配器 |
| F02 | Alert 身份 = `cluster_id` + fingerprint；IngestCredential ≠ OIDC | BR-016、BR-020–023 | 独立 ingest 面；幂等 upsert；缺 `cluster_id` 拒绝 |
| F03 | 一期仅人点调查；二期仅 `severity` 精确等于 `critical` 才自动开查 | BR-030、BR-070–074 | 调查创建是命令；自动开查有并发闸与审计，不排队 |
| F04 | Claim 必须绑定服务端 Citation；零 citation ⇒ `inconclusive` | BR-031–032、§5.2 | Citation 铸造在控制面；模型不得发 id |
| F05 | 只读工具白名单；8 LLM / 12 工具 / 150s；每用户每天 30 次人工调查 | BR-033、BR-040–042 | 调查是有界异步任务 |
| F06 | 一期无外部写；二期仅人批 `restart`/`scale`；模型不可自批；写结果不是 Citation | BR-044、BR-080–088 | 审核账本 = `ApprovalRequest`，不是通用公文流 |
| F07 | 知识库上传、无 `alert_id` 对话、Playwright：永不做 | §9、BR-089 | **不设计用户文件上传**；证据 = ToolCall 截断副本 |
| F08 | 角色仅 SRE / 开发 on-call / 唯一 break-glass；开发不可见 `unscoped` | §4、BR-017、BR-022 | 授权按标签绑定；前端隐藏不是安全边界 |
| F09 | 保留：调查/ToolCall/Citation/Alert 90 天；AuditEvent 365 天 | BR-060–062 | 到期删除作业 |
| F10 | 页面 PG-001–012；TF-01–17；`completed` 与 `inconclusive` 必须可区分 | 交互全文 | API 状态枚举与页态一一映射 |
| F11 | 实现技术栈已冻结：FastAPI / uv / PostgreSQL 18 / React 19 / Vite 8 | ADR-0008 Accepted；[`tech_stack.md`](./tech_stack.md) | 版本与 CVE 审计见该文；系统架构其余部分仍 Draft |

### 2.2 假设（不得升格为已批准产品结论）

| ID | 假设 | 来源 | 架构影响 |
|---|---|---|---|
| A01 | 现网已有 K8s / Prom / AM / xMatters | 调研 + BR | HuntAI 只做接入层 |
| A02 | 创建后调查不可取消 | 交互 `[ASSUMPTION]` | 无 cancel API |
| A03 | 同集群已有 `running` ScanRun 则拒绝再发起 | 交互 `[ASSUMPTION]` | 扫描互斥 |
| A04 | LangGraph 只做调查图，不做业务控制面 | README `[ASSUMPTION]`；本阶段 ADR-0002 | 控制面持有入库/ACL/配额/扫描/审批 |
| A05 | SQLite 不得承担审批 / 审计 / 恢复 | README `[ASSUMPTION]`；ADR-0005 | 持久化默认 PostgreSQL |
| A06 | 150s 只计 LLM+工具；pending 不消耗墙钟；Investigation 保持 `running` 直至审批结束或超时 | 用户批准 D2；已写入 **BR-040 / BR-090** | 超时秒数仍 `[TBD-BIZ]` |
| A07 | 仅 `running` 可「继续查」；结束后人对同一 Alert 再开新的 `trigger=human` | 交互 `[ASSUMPTION]` | 无在终态上追加工具循环的 API |
| A08 | 配额自然日界 `Asia/Shanghai` | 维护人「按建议」；BR-041 | 若公司 IdP 时区不同再改 |

### 2.3 冲突（fail-close）

| ID | 冲突 | 处置 |
|---|---|---|
| C1 | 当次指令要 `architecture.md` + `00`–`04` + `adr/`；AGENTS.md 约定 `architecture_spec.md`、`frontend_design_spec-v1.0.md`、`frontend_backend_boundary_spec-v1.0.md` | 用户已批：集成正文双文件名；**前端设计规范等 Stage 7 API 之后再写**；写边界原则 Draft |
| C2 | 当次文件名连字符 vs `project_rules` snake_case | 用户已批：专题用 snake_case |
| C3 | BR-040 150s 结束 vs BR-085 离开 `running` 则 pending 作废 | 已写入 BR-040 / BR-090：墙钟不含人批等待；超时数字仍 `[TBD-BIZ]` |
| C4 | `.env.example` `ALERT_WEBHOOK_URL` vs BR-044 禁止出站呼叫/IM/邮件 | 架构**不**把该键当产品通知通道。键去留 `[TBD-INFRA]`，本阶段不改冻结的 `.env.example` |
| C5 | 根 README 阶段表仍写 01–04「待启动」 | 以磁盘文件为准；不在本次改正文 |

### 2.4 缺失 / 待确认（禁止虚构）

| 项 | 标记 | 架构策略 |
|---|---|---|
| IdP 产品名 | `[TBD-BIZ]` | OIDC 适配器可插拔，文档不写商标 |
| 无 LLM 时「调查」拒绝还是只跑工具 | **已拍板 BR-053**：只跑工具 | — |
| 人点并发 5 是否硬拒绝 | **已拍板 BR-054**：不硬拒绝 | — |
| 组织 token 闸是否阻断 `trigger=auto` | **已拍板 BR-042**：阻断 | — |
| 跨用户是否可看他人 Investigation | **已拍板 BR-055**：能见 Alert 则可只读 | — |
| 日额拒绝是否占次数 | **已拍板 BR-041**：不占 | 临时放行额度/时限仍 `[TBD-BIZ]` |
| 脱敏清单、截断字节、出域默认 | 无清单不得出域；默认可逆变脱敏后 BYOK，预留禁出域 | 清单与字节、网关地址仍 `[TBD-BIZ]` |
| 深链 URL path | `[TBD-FE]` | 只约束参数 `cluster_id` + fingerprint |
| webhook P95、可用性百分比、删除时限 | `[TBD-BIZ]` | 禁止写「高可用」 |
| 原型 / Gate 1 | 缺失 | 不阻塞 |
| 证据落对象存储 vs 仅 PostgreSQL | `[TBD-INFRA]` | 一期默认库内截断文本 |
| `frontend_design_spec-v1.0.md` | 维护人 2026-09-14：**先不写**，等 Stage 7 API 契约后再做前端架构 | Stage 6 该项未完成，不阻塞后端 Draft |

### 2.5 提示词能力映射（防止设计跑偏）

| 提示词用语 | 本产品对应 | 不对应 |
|---|---|---|
| 文件上传 | ToolCall 输出副本 / 截断证据（BR-043） | 知识库上传 UI（§9.25） |
| 任务状态 | `Investigation`、`ScanRun` | 通用 job 平台 |
| 审核对象 / 审核记录 | 二期 `WriteAction` + `ApprovalRequest` + `AuditEvent` | RCA Claim 人工审改（PG-006 禁止编辑 Claim） |
| MCP | 仓库编码助手文档 MCP | 产品调查工具总线 |

---

## 3. 官方技术调研

规则：每条含问题、官方链接、结论、影响、采用或不采用理由。调研日 2026-09-14。

### R1 — 产品运行时是否引入 MCP

- **问题**: HuntAI 调查需要调用 kube / Prom /（二期）Loki。是否应把这些工具暴露为 MCP Server，由 Host/Client 调用？
- **官方链接**:
  - 规范总览：<https://modelcontextprotocol.io/specification/2025-11-25>
  - 架构（Host / Client / Server，tools / resources / prompts）：<https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture>
  - 安全最佳实践（confused deputy、工具为任意代码执行、须用户同意）：<https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices>
- **结论**: MCP 是 **Host（AI 应用）— Client（每服务器一条连接）— Server（提供 tools/resources/prompts）** 的上下文交换协议。Server 的 Tools 是可执行函数；规范要求 Host 在调用工具前取得明确用户同意，并将工具描述视为不可信（除非 Server 受信任）。MCP **不规定** 如何做鉴权账本、配额、citation 铸造或审批。远程传输还引入 OAuth / confused deputy 攻击面。
- **对本项目的影响**: 调查工具集合已由 BR-033 / BR-075 / BR-080 **写死白名单**，调用方是 HuntAI 控制面而非终端用户逐次授权每个 tool。权限、namespace 注入、脱敏、citation 必须在工具适配器内强制，不能依赖 MCP 注解。
- **不采用（产品运行时）理由**:
  1. 无 BR 要求「把集群能力做成 MCP Server」。
  2. 调研报告明确「不纳入立项：MCP 全量 CRUD」。
  3. 引入 Host/Client/Server 会把允许列表从代码策略改成协议发现，Holmes #2228 已证明 allowlist 失效风险。
  4. 仓库 `.mcp.json` 仅为 **LangChain 文档** MCP，服务编码 Agent，与运行时分离（红线：两份 MCP 清单须一致，本阶段不改）。
- **替代**: 一等 Python 工具适配器，由调查运行时按白名单调用。

### R2 — 人批是否采用 LangGraph HITL（interrupt / checkpoint / resume）

- **问题**: 二期 `ApprovalRequest` 是否用 LangGraph `interrupt()` 暂停图、checkpoint 持久化、`Command(resume=…)` 恢复，并让图执行写操作？
- **官方链接**:
  - Interrupts：<https://docs.langchain.com/oss/python/langgraph/interrupts>
  - Checkpointers：<https://docs.langchain.com/oss/python/langgraph/checkpointers>
  - Persistence（生产须持久 checkpointer；`InMemorySaver` 重启即丢）：<https://docs.langchain.com/oss/python/langgraph/persistence>
  - 类型参考（resume 从**中断节点开头重跑**）：<https://reference.langchain.com/python/langgraph/types/interrupt>
- **结论**:
  1. `interrupt()` 会保存图状态并 **无限等待** 直到 resume（官方原文：waits indefinitely）。
  2. Resume 必须同一 `thread_id`；**节点从开头重执行**，`interrupt` 之前的副作用必须幂等，写操作应放在 interrupt **之后**或独立节点。
  3. HITL 需要 checkpointer；生产不可用内存实现。
- **对本项目的影响**: 与 BR-082（模型不可执行写）、BR-086（WriteIdentity 分离）、BR-085（离开 `running` 则 void）、BR-040（有界墙钟）冲突，若把审批账本和图执行绑在一起：无限等待 vs 墙钟；resume 重跑可能导致重复写；图内执行写会让模型工具路径碰到 WriteIdentity。
- **不采用（审批账本 / 写执行）理由**: 审批的权威记录必须是 `ApprovalRequest`（控制面 PostgreSQL），执行只走 confirm 命令 + WriteIdentity。LangGraph 图 **不得** 调用写动词。
- **LangGraph 仍采用的范围**: 仅调查图（Skill、只读工具循环、成本闸计数、给控制面的 Claim 草案）。见 ADR-0002、ADR-0004。
- **可选、默认关闭**: 用 interrupt 暂停「剩余 LLM 循环」——无必要，因为 RCA 与人批并行更符合「写结果不是 Citation」。

### R3 — 普通上传 / 任务查询 / 审核查询是否构成引入 MCP 或 LangGraph 的理由

- **问题**: 提示词要求「普通文件上传、任务状态或审核查询不默认引入」。
- **官方链接**: 同 R1、R2（MCP/LangGraph 均为通用编排协议，不提供 HuntAI 领域语义）。
- **结论**: 本产品无用户上传；任务查询是 `Investigation`/`ScanRun` 的 GET；审核查询是 `ApprovalRequest` 的 GET。这些是领域 CRUD + 状态机，不需要 MCP 资源或 LangGraph interrupt。
- **不采用理由**: 无业务缺口需要这些框架才能表达。

---

## 4. 阶段二文档清单（本目录）

| 文件 | 作者角色 | 状态 |
|---|---|---|
| `00_research_and_input_traceability.md` | 调研 | 本文 Draft |
| `01_domain_and_service_architecture.md` | Agent A | Draft |
| `02_api_workflow_and_review.md` | Agent B | Draft |
| `03_security_reliability_and_operations.md` | Agent C | Draft |
| `04_architecture_review.md` | Agent D | Draft |
| `architecture.md` | Lead 集成入口 | Draft；与 spec 同文 |
| `architecture_spec.md` | Stage 6 约定名 | Draft；与 architecture.md 同文 |
| `frontend_backend_boundary_spec-v1.0.md` | Lead + B | Draft 原则，非 OpenAPI |
| `adr/0001`–`0007` | Lead | Proposed |
| `frontend_design_spec-v1.0.md` | — | **延后**：等 Stage 7 API 契约后再写 |

---

## 5. 开放问题（需用户确认，禁止在实现里猜）

见集成稿 §开放问题。本文件不重复展开数字类 TBD。
