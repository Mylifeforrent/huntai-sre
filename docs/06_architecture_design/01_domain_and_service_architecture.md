# 领域与服务架构

- **Status**: Draft
- **日期**: 2026-09-14
- **角色**: Agent A（领域与服务架构师）
- **权威输入**: [`../03_problem_modeling/business_model.md`](../03_problem_modeling/business_model.md)；协作线索 [`../04_interaction_design/interaction_design_summary.md`](../04_interaction_design/interaction_design_summary.md)
- **追溯**: [`00_research_and_input_traceability.md`](./00_research_and_input_traceability.md)
- **非目标**: 不写表字段、不写 OpenAPI、不写部署清单、不生成代码。

---

## 1. 系统边界

HuntAI SRE 是接在现网 Kubernetes / Prometheus / Alertmanager / xMatters **之上**的可信 AI 运维层（BR-001）。本系统：

| 在边界内 | 在边界外（权威仍在对方） |
|---|---|
| Alert 入库副本、调查、规则扫描、Citation、审计、人批写修复（二期） | Prometheus 指标存储与规则；Alertmanager 路由与 silence；xMatters 排班/呼叫 |
| 只读 kube / Prom /（二期）Loki 查询 | 集群控制面、secrets、任意 apply/delete/exec |
| Web API + 调查 worker | 浏览器渲染（前端）；编码助手 MCP |

**信任边界**

```text
[Alertmanager] --IngestCredential--> [Ingest]
[IdP OIDC]    --用户会话----------> [API / 控制面]
[break-glass] --本地口令----------> [API / 控制面]
[控制面]      --只读 SA-----------> [各 ClusterSource 的 kube-apiserver]
[控制面]      --Prom 只读---------> [各集群 Prometheus]
[控制面]      --LokiCredential---> [各集群 Loki]          〔二期〕
[控制面]      --WriteIdentity----> [kube 仅 restart/scale] 〔二期〕
[调查 worker] --脱敏后 BYOK------> [LLM 网关]              禁出域则不连
```

一期对接 2–5 个集群，每集群一套 Prom + 一套 AM（BR-004、NFR-013）。xMatters **无 API 集成**（§2.4）：只保证深链 URL。

---

## 2. 运行时形态与模块

**推荐**: 模块化单体（一个 FastAPI 进程对外 + 一个调查/扫描 worker 进程）。理由：NFR-010 一期 20 人；单人运维；事务与审计同库。见 ADR-0001。

禁止把控制面状态机放进 LangGraph。见 ADR-0002。

### 2.1 模块职责

| 模块 | 阶段 | 职责 | 不负责 |
|---|---|---|---|
| **identity** | 一期 | OIDC 会话；恰好一个 break-glass；角色；TeamBinding 读写 | IdP 本身；用户自助勾选团队 |
| **ingest** | 一期 | 校验 IngestCredential；解析 `cluster_id`；fingerprint（BR-020）；Alert upsert；打 `unscoped` | 创建 Investigation（一期）；改 xMatters receiver |
| **alert_query** | 一期 | 按角色列出/打开 Alert；深链查找；写「看过」AuditEvent；RelatedLink | silence / ack |
| **investigation_control** | 一期 | 创建/查询 Investigation；日额与组织闸；人点并发；终态判定；Citation 铸造 | 直接调 kube；模型自报完成 |
| **investigation_runtime** | 一期 | LangGraph 调查图：拉 Skill、只读工具循环、成本闸计数、产出 Claim 草案与 ToolCall 记录 | ACL、配额、写集群、MCP |
| **evidence** | 一期 | 持久化 ToolCall 输入/输出副本、截断标记；供 Citation 指向 | 用户上传；把 RelatedLink 当证据 |
| **scan** | 一期 | 手动 ScanRun；规则分析器（零 LLM）；Finding 快照 | Finding ack/close；开发触发 |
| **catalog** | 一期 | ClusterSource、IngestCredential、Prom 只读账号、Skill 映射 | 上传 kubeconfig |
| **audit_retention** | 一期 | append-only AuditEvent；90/365 天删除 | 独立审计台产品 |
| **auto_investigate** | 二期 | BR-070–074：critical 自动开查、fingerprint 去重、并发 ≤5 | 缺键放行；排队补开 |
| **loki_tool** | 二期 | LogQL 适配器 + 开发 namespace 注入 | Elastic；UI 扒页 |
| **remediation** | 二期 | WriteAction + ApprovalRequest；confirm 走 WriteIdentity | 模型自批；delete/apply/exec |

Worker 运行 `investigation_runtime` 与 `scan`（及二期 `auto_investigate` 触发的同一调查图）。HTTP API 只在控制面。

### 2.2 一期可部署切片 vs 二期边界

一期进程 **可以包含** 二期模块的空实现（返回「能力未开启」），但：

- 不得暴露二期 API 作为一期验收；
- 不得在一期 UI 契约中出现自动开查徽章、LogQL、ApprovalRequest、`env`/`write_remediation`（交互 0.3）。

二期打开 = 配置与功能开关，不是拆新微服务。

---

## 3. 核心领域模型

实体以 Business Model §3 为准，此处只补**生命周期与不变量**。不新增 Incident、Mailbox、无 `alert_id` 的 ChatSession。

### 3.1 关系（逻辑）

```text
Organization 1 ── * User
Organization 1 ── * ClusterSource
User * ── * TeamBinding（标签值集合）
ClusterSource 1 ── * IngestCredential
ClusterSource 1 ── * Alert
Alert 1 ── * Investigation
Investigation 1 ── * ToolCall
Investigation 1 ── * Claim
Claim * ── * Citation（每条 Claim ≥1）
Citation → 本调查内已成功的只读 ToolCall
ClusterSource 1 ── * ScanRun ── * Finding
Investigation 1 ── * WriteAction ── 0..1 ApprovalRequest   〔二期〕
ClusterSource 0..1 WriteIdentity / LokiCredential          〔二期〕
```

基数：Organization = 1（BR-002）。Alert 不跨 `cluster_id` 合并（BR-020）。

### 3.2 关键不变量

| # | 不变量 | 来源 |
|---|---|---|
| I1 | Citation.id 仅控制面铸造；不得来自模型输出 | BR-031 |
| I2 | Citation 不得指向失败/未执行/他次调查/WriteAction 结果/RelatedLink/user-supplied | BR-031、BR-034、BR-037、BR-088 |
| I3 | Citation 数 = 0 时 Investigation 不得为 `completed` | BR-032 |
| I4 | `auto-investigator` 不可登录、不可 confirm、不计入用户日额 | BR-073 |
| I5 | 写工具仅当 `write_remediation=开` **且** 已配 WriteIdentity | BR-081 |
| I6 | `env` 未填视为 `production`；不得用集群名推断 | BR-087 |
| I7 | 密钥、未脱敏原文不得进入调查库、Citation 副本、应用日志 | BR-052 |

### 3.3 状态机（与业务文档一致，不发明新主状态）

**Alert**（§5.1）: `firing` ↔ `resolved`；同 fingerprint 更新字段。`unscoped` 是标签不是状态。

**Investigation**（§5.2）:

```text
（鉴权/额度失败）→ 不落 running
人工或二期自动 → running
running → completed   （≥1 Claim 且每条 ≥1 Citation 且未触闸）
running → inconclusive（零 Citation 或 LLM/工具/墙钟触闸）
```

**控制面补充（BR-090）**: 调查图的 LLM+工具循环结束后，若仍存在 `pending` ApprovalRequest，**主状态保持 `running`**，直到全部 pending 变为 confirmed/rejected/void 或超时。墙钟计数在 pending 期间暂停。超时秒数 `[TBD-BIZ]`。界面 L1 不得表现为「仍在分析」。

**ScanRun**（§5.3）: `running` → `completed` | `scan_failed`。Finding 随 completed 冻结。

**ApprovalRequest**（§5.4，二期）: `pending` → `confirmed` | `rejected` | `void`。

界面必须区分 `completed` 与 `inconclusive`（G1、IX-FB-04）。无「取消调查」命令（A02）。

---

## 4. 命令、查询、异步任务、事件

### 4.1 命令（改变状态）

| 命令 | 阶段 | 发起者 | 主要规则 |
|---|---|---|---|
| UpsertAlert | 一期 | IngestCredential | fingerprint 幂等；一期不建 Investigation |
| CreateInvestigation | 一期 | 可见该 Alert 的人 | 日额、组织闸；**不**用人点并发硬拒绝（BR-054）；`trigger=human`。无 LLM 时仍创建并只跑工具（BR-053） |
| RecordAlertSeen | 一期 | 打开 PG-005 的用户 | 只写 HuntAI 审计，不同步 AM/xMatters |
| AddRelatedLink | 一期 | 可见该 Alert 的用户 | URL 句法；host 允许列表 `[TBD-BIZ]` |
| StartScanRun | 一期 | SRE / break-glass | 零 LLM；同集群 running 互斥（A03） |
| GrantDailyQuotaOverride | 一期 | SRE / break-glass | 必须指定用户；额度/时限 `[TBD-BIZ]` |
| SetOrgTokenGate | 一期 | SRE / break-glass | 未配置不得视为无限；关闸确认 |
| UpdateTeamBinding | 一期 | SRE / break-glass | 写 AuditEvent；禁止自助 |
| RegisterClusterSource | 一期 | SRE / break-glass | 上限 5；无 kubeconfig 上传 |
| AutoCreateInvestigation | 二期 | 系统 | BR-070–072；失败跳过+审计 |
| ProposeWriteAction | 二期 | 调查图经控制面 | 只生成 pending + preview |
| ConfirmWriteAction / RejectWriteAction | 二期 | 有权登录用户 | R2；执行用 WriteIdentity；IX-DUP-04 |
| ContinueInvestigation | 二期 | 能打开该调查的人 | 仅 `running`（A07）；仍受 8/12/150s |

创建失败 **不** 留下 `running` 行（EX-05.1、业务 §5.2 `rejected` 语义 = 未落库）。拒绝与接口失败 **不占** 当日次数（BR-041）。

### 4.2 查询

告警列表（角色过滤）、深链定位、调查详情（含 ToolCall 时间线与 Citation）、ScanRun/Finding、配置只读。开发的读模型在服务端过滤，禁止「先全量再让 UI 藏」。

深链：参数合法但无行 → 对调用方表现为未入库（PG-004）；有行但不可见 → 与「不存在」的探测差异 `[TBD-BIZ]`（交互倾向：不可见走 No Permission）。

### 4.3 异步任务

| 任务 | 触发 | 有界性 | 进度 |
|---|---|---|---|
| 调查图运行 | Create / AutoCreate | 8/12/150s（150s 定义见 A06） | 控制面投影 ToolCall 与闸计数 |
| 规则扫描 | 手动 / 二期每 60 分钟 | 集群不可达 → `scan_failed` | ScanRun 状态 |
| 保留删除 | 调度 | NFR-033 时限 `[TBD-BIZ]` | 审计删除批次 |
| 二期写执行 | confirm | 失败禁止 fallback SA/kubeconfig | ApprovalRequest + AuditEvent |

前端关页不取消调查（EX-06.5）。无产品级「通知推送」（BR-044）；进度靠查询或可选 SSE（见 02 文档）。

### 4.4 领域事件（进程内；非对外总线）

至少：`AlertUpserted`、`InvestigationStarted`、`InvestigationTerminal`、`ScanCompleted`、`QuotaOverrideGranted`、`TeamBindingChanged`、`AutoInvestigateSkipped`、`WriteActionConfirmed`。订阅者写 AuditEvent 或触发二期自动开查。 **不** 引入外部 Kafka/NATS，除非未来 NFR 要求（当前无）。

一致性：Alert upsert、配额扣减、自动开查判定与审计必须在同一数据库事务或等价的「先持久再投递」；调查图与控制面之间以 Investigation id 为关联，图失败不得把状态标 `completed`。

---

## 5. 「文件」、任务、审核的服务归属

| 能力 | 归属模块 | 说明 |
|---|---|---|
| 证据字节 | evidence + investigation_control | 截断后的 ToolCall 副本；Citation 指向该副本。阈值 `[TBD-BIZ]` |
| 用户文件上传 | **不实现** | §9.25、BR-089 |
| 任务状态 | investigation_control / scan | GET 资源状态；不是通用 job API |
| 审核对象 | remediation（二期） | 一条 WriteAction 对应一条 ApprovalRequest |
| 审核记录 | audit_retention | confirm/reject/void 及配置变更 |

---

## 6. 幂等、审计、失败恢复、并发

### 6.1 幂等

| 操作 | 键 | 行为 |
|---|---|---|
| AM webhook | `cluster_id` + fingerprint | upsert；重复 firing 不新建 Alert，一期不新建调查 |
| 人点调查连点 | 客户端禁用 + 服务端幂等键（请求 id）`[ASSUMPTION]` | 只创建一条（IX-DUP-02）；同一 Alert **允许** 另一次带新幂等键的人工调查 |
| 自动开查 | fingerprint + 本次 firing 窗口 | 最多 1 条 `trigger=auto`（BR-071） |
| 扫描连点 | 集群 + running 互斥 | A03 |
| confirm 写 | ApprovalRequest id | 只执行一次（IX-DUP-04） |

Webhook 重试必须安全：已 resolved 再 firing 是合法状态回切（§5.1），不是重复创建。

### 6.2 审计

AuditEvent append-only，覆盖：登录、绑定变更、开调查、扫描、数据源配置、额度放行、看过（MVP-18）；二期加跳过自动开查、写确认。禁止把密钥、Prompt 原文、未脱敏日志写入审计载荷（BR-052、IX-FB-03）。

### 6.3 失败恢复

| 失败 | 恢复 |
|---|---|
| 调查 worker 崩溃 | Investigation 仍 `running`；worker 按 id 续跑或在墙钟耗尽后标 `inconclusive`（成本）。**禁止**无 citation 标完成 |
| 单次只读工具失败 | 该次无 Citation；循环可继续直至闸（EX-06.3） |
| 只读 SA 不可达 | 工具失败；生产禁止 fallback kubeconfig（BR-048） |
| 扫描失败 | `scan_failed`；允许新开 ScanRun |
| 写身份失败 | 不执行；禁止 fallback（BR-086） |
| LLM 不可用 / 禁出域 | BR-053：仍开调查、只跑工具；不得声称已完成 RCA；列表与扫描仍可用（G3） |

LangGraph checkpointer：若采用图持久化，必须用 PostgreSQL 等耐久实现，禁止 `InMemorySaver` 上生产（官方 persistence 文档）。Checkpointer **不是** ApprovalRequest 账本。

### 6.4 并发冲突

| 资源 | 冲突 | 策略 |
|---|---|---|
| 全组织 auto `running` | NFR-017 ≤5 | 事务内计数；满则跳过+审计，不排队。组织闸关闭/未配置同样跳过（BR-042） |
| 人点 `running` | NFR-011 规划值 | **不硬拒绝**（BR-054） |
| TeamBinding 与调查中读模型 | 映射中途变更 | 调查开始时固化可见性快照 `[ASSUMPTION]`；新请求用新映射 |
| 同 Investigation 双 confirm | 见 6.1 | 行级锁 ApprovalRequest |
| Alert 与自动开查 | upsert 同时判定 BR-070 | 同一事务：先 upsert Alert，再条件插入 Investigation |

---

## 7. 调查运行时（LangGraph）契约

图的输入：Investigation id、Alert 快照、触发者（User 或 `auto-investigator`）、ClusterSource 只读连接、匹配到的 Skill 文本（可空）。

图允许做：按白名单调用只读适配器；把原始工具结果交给 evidence（截断）；把 **未带 Citation id** 的断言草案交给控制面。

图禁止做：决定 HTTP 状态；扣配额；铸造 Citation id；调用写动词；跳过 citation 规则（含 auto）；把 RelatedLink 当证据。

成本闸由控制面或运行时拦截器强制：超限停止并请求将 Investigation 标 `inconclusive`（成本）。Skill 空 → 调查继续，标记「未加载组织约束」（BR-038）。

二期：只读白名单增加 Loki；写提议通过控制面命令 `ProposeWriteAction`，不在图内执行。

---

## 8. 本专题开放问题

- 调查开始时是否固化 TeamBinding 快照（A 假设是）。
- 人点调查幂等键是 Header 还是仅靠前端禁用（前端禁用不是安全边界）。
- 证据超阈值：截断入库 vs 对象存储 `[TBD-INFRA]`。
- pending 人批等待超时秒数 `[TBD-BIZ]`。
