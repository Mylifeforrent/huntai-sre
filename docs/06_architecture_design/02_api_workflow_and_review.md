# 接口、任务与审核工作流

- **Status**: Draft
- **日期**: 2026-09-14
- **角色**: Agent B（接口、任务与审核工作流架构师）
- **权威输入**: Business Model §2/§4/§5；交互 TF/EX/IX-*、页面状态矩阵
- **协作**: [`01_domain_and_service_architecture.md`](./01_domain_and_service_architecture.md)
- **非目标**: 完整 OpenAPI 与字段表（Stage 7）；前端视觉；实现代码。

---

## 1. 前后端接口边界

### 1.1 原则

| 原则 | 说明 | 来源 |
|---|---|---|
| 浏览器只谈 HTTP API | 前端不直连 kube / Prom / Loki / IdP token 换集群凭证 | BR-046–048、MVP-20 |
| 权限以后端为准 | 隐藏导航不是安全边界 | 交互 §3 开篇、project_rules 4.2 |
| 状态权威在服务端 | `Investigation.status`、`ScanRun.status`、`ApprovalRequest.status` 由控制面写入 | §5 |
| 同步命令返回资源 id | 创建调查成功以进入 PG-006 为准（IX-FB-01），API 必须返回 Investigation id |
| 错误不泄密 | 无栈、SQL、密钥、Prompt、未脱敏日志 | IX-FB-03、BR-052 |
| 路径风格 | `/api/v1/` + 小写连字符复数名词 | `project_rules` §1.3；**具体路径 Stage 7 冻结** |

本阶段只锁定**资源与动词语义**，不锁 URL 字符串（深链 path 仍 `[TBD-FE]`）。深链查询参数必须能表达 `cluster_id` + fingerprint（BR-051）。

### 1.2 请求 / 响应

- JSON；时间戳 UTC。
- 写命令：需要登录会话（OIDC 或 break-glass），CSRF 策略 `[TBD-FE]`/`[TBD-BE]`。
- Ingest webhook：**不**使用用户会话；使用 IngestCredential（BR-016）。
- 列表过滤在服务端执行（集群、firing/resolved、角色范围）。不做全文搜索（交互：alertname 搜索 `[TBD-BIZ]`，本期不做）。
- 幂等：可能重复提交的写命令接受 `Idempotency-Key` `[ASSUMPTION]`（调查创建、扫描发起、confirm）。前端禁用不能替代该键。

### 1.3 错误模型

统一 envelope `[ASSUMPTION]`：`error.code`、`error.message`（面向用户的短句）、无 `details` 内部结构。HTTP 状态与业务码分开。

| 情境 | HTTP（建议） | 业务码（逻辑名，Stage 7 编号） | 页态 |
|---|---|---|---|
| 未登录 | 401 | `unauthenticated` | 回 PG-001（EX-01.1/01.3） |
| 角色/标签不可见 | 403 | `forbidden` | No Permission；深链不可见不泄露 alertname（EX-03.2、IX-PERM-02） |
| 未入库深链 | 404 | `alert_not_ingested` | PG-004 固定文案 |
| 深链缺参数 | 400 | `deep_link_incomplete` | PG-004 Error（EX-03.3） |
| 日额用尽 | 409 或 429 | `daily_investigation_quota` | Success-Failure（EX-05.2）；**不占**次数（BR-041） |
| 组织关闸 / 闸未配置 | 409 | `org_gate_closed` / `org_gate_unconfigured` | EX-05.3；二期 auto 同样跳过（BR-042） |
| 人点并发超过规划值 | 201 | （无错误码） | 不拒绝（BR-054）；正常创建 |
| 无 LLM / 禁出域 | 201 | 资源上标记 `llm_unavailable` | EX-05.6：只跑工具；禁止返回 `completed` RCA |
| 校验失败 | 422 | `validation` | 表单内联 |
| 系统失败 | 500 | `internal` | Error；无内部细节 |

`completed` / `inconclusive` / `scan_failed` / `void` 是**资源状态**，不是 HTTP 错误。

### 1.4 权限边界（API 强制）

映射 Business Model §4 与 IX-PERM。抽样：

| API 类 | SRE / break-glass | 开发 on-call | Ingest |
|---|---|---|---|
| 列出 Alert | 全部含 unscoped | TeamBinding 匹配 | 否 |
| 创建调查 | 可见即可 | 仅可见 Alert | 否 |
| 扫描触发 | 是 | 否（二期也否） | 否 |
| 扫描结果 | 全部 Finding | 二期：namespace 相交只读 | 否 |
| 设置类 | 是 | 否 | 否 |
| webhook upsert | 否 | 否 | 是 |
| 二期 confirm 写 | 任意 env | 仅 `non_production` 且 namespace ∈ 绑定 | 否 |
| 二期 Loki | 调查内不限 ns | 强制注入 namespace；无绑定则拒 | 否 |

`auto-investigator` 无任何 HTTP 主体。break-glass 权限同 SRE，但会话必须可被审计区分（横幅是 UI，审计是后端）。

跨用户读他人 Investigation：BR-055——能见 Alert 则可只读其调查。

---

## 2. 同步、异步、进度、通知、重试

### 2.1 调查

```text
POST 创建 ──同步──► 201 + investigation_id, status=running
                 └── 入队 worker
GET  详情 ──同步──► 当前状态、闸计数、ToolCall、Claim/Citation
GET  事件流（可选）──SSE──► 工具增量  〔非必须；不可替代 GET〕
```

- 创建在鉴权、配额、闸校验 **通过并落库后** 才返回成功。失败不占 `running`，也 **不占** 日额（BR-041）。
- Worker 在用户关页后继续，直到闸或终态（EX-06.5）。
- 进度字段至少：`status`、`llm_calls_used`、`tool_calls_used`、`deadline_at` 或已用墙钟、`skill_loaded`、〔二期〕`awaiting_write_approval`。
- **通知**: 不做 IM/邮件/呼叫（BR-044）。前端用轮询或 SSE。轮询间隔 `[TBD-FE]`。
- **重试**: 客户端对 GET 可重试；对 POST 创建必须带幂等键。Worker 对只读工具失败不盲目整图重放（避免重复 LLM 计数）；单工具失败记一行后继续。
- 墙钟：只计 LLM+工具执行（BR-040）。存在 pending 审批时主状态保持 `running`，墙钟暂停（BR-090）。

无 LLM 时：列表仍可用（BR-026）。点「调查」**仍创建**，只跑工具（BR-053）；禁止返回假装 `completed` 的 RCA。

### 2.2 扫描

POST 启动 → ScanRun `running` → GET 至 `completed`/`scan_failed`。零 LLM。同集群已有 running → 409（A03）。二期调度器每 60 分钟对每个 ClusterSource 发同一命令，`trigger=scheduled`；开发不可 POST。

### 2.3 Webhook

Alertmanager POST：校验凭证 → upsert → 202/200。缺凭证/缺 cluster_id → 4xx，不入库。**一期**响应时不得创建 Investigation。二期在同一请求的事务后半段尝试 AutoCreate；失败跳过不得导致 Alert 回滚（BR-072：Alert 仍在）。

重试：AM 会重复 POST；upsert 幂等。

---

## 3. 审核流程（二期；一期 API 不暴露）

对应 TF-15、§5.4、BR-080–088。不设「审批中心」资源（交互 1.1）。

| 项 | 设计 |
|---|---|
| 对象 | `WriteAction`（verb=`restart`\|`scale`）+ `ApprovalRequest` |
| 角色 | 提议：调查图经控制面。确认/拒绝：登录用户且过 R2。`auto-investigator` 不能 confirm |
| 操作 | 生成 preview（缺 preview 不得出现确认）→ confirm 第二次点击 → 或 reject |
| 撤回 | 无「撤回已执行」。reject = 不执行。离开调查 `running` → `void`（BR-085） |
| 退回 | 无多级审批。权限/上限失败 → 拒绝执行（EX-15.3 保持 pending 或转 rejected，以后端一次判定为准，**推荐直接 `rejected`** 以免重复 confirm） |
| 修改 | 不提供改 preview 后再批；模型需新提议则新 WriteAction `[ASSUMPTION]` |
| 重新提交 | 人对同一 Alert 再开调查可再提议；void 的请求不可复活 |
| 超时 | pending 等待上限 `[TBD-BIZ]`；到点 `void`，再允许 Investigation 进入终态 |
| 审计 | confirm/reject/void/执行结果写 AuditEvent；结果不可 Citation |

**不采用 LangGraph HITL 作为审批账本或写执行路径**（调研 R2、ADR-0004）。confirm 是控制面命令：校验 → WriteIdentity 执行 → 更新 ApprovalRequest。图不 resume 去 kubectl。

一期相对二期：API 与错误码表可预留命名空间，但一期客户端不得调用；调用返回 404 或 `capability_not_enabled`，避免「即将上线」语义（交互 0.3）。

---

## 4. 前端页态 ↔ 后端状态

| 页面 | 后端依据 | Default | Empty / No Result | No Permission | Success-Failure |
|---|---|---|---|---|---|
| PG-001 | 无会话 | 登录页 | — | — | IdP 失败 |
| PG-002 | break-glass 挑战 | 表单 | — | — | 凭证错误（不区分用户是否存在） |
| PG-003 | Alert 列表查询 | 有可见行 | 无筛选 0 条 / 筛选 0 条 | 角色异常 | — |
| PG-004 | 深链查找 404 `alert_not_ingested` | 固定文案 | 与 Default 同义 | 不可见不得落本页 | — |
| PG-005 | Alert GET | 资源存在且可见 | 无此页（走 004） | 403 | 额度/闸拒绝创建调查 |
| PG-006 | Investigation GET | `running` 或终态已渲染 | 保留期满删除 | 403 | `completed` vs `inconclusive` 必须分徽章；二期 pending/confirmed/rejected/void |
| PG-007 | ScanRun 列表 | 有历史 | 无运行 / 开发无相交 Finding | 一期非 SRE | 发起失败 |
| PG-008 | ScanRun GET | completed 且有可见 Finding | completed 且 0 条可见 | 403 | `scan_failed` |
| PG-009–012 | 配置资源 | 列表/表单 | 0 集群等 | 开发 403 | 保存业务失败 |

调查终态映射（IX-FB-04）：

| `Investigation.status` | 前端 L1 |
|---|---|
| `running` | 时间线 + 闸；不是 Success-Failure |
| `completed` | 「调查完成」类；Claim+Citation |
| `inconclusive` | 禁止「根因是…」；原因无引用或成本闸 |

二期 PG-006 人批区跟随 `ApprovalRequest.status`，从属于仍为 `running` 的调查，不得单独升级成「根因已修复」（IX-FB-09）。

---

## 5. MCP 与 LangGraph HITL（本专题结论）

资料明确存在：**Agent 工作流**（调查工具循环）、**人工审批**（二期 WriteAction）、**模型工具调用**（白名单适配器）。因此按提示词要求做了评估（见 `00` R1/R2），而不是跳过。

| 技术 | 结论 |
|---|---|
| 产品运行时 MCP | **不采用**。工具不是第三方 MCP Server；权限与 citation 必须在进程内强制。仓库文档 MCP 保持只给编码 Agent。 |
| LangGraph | **采用，仅调查图**。控制面状态机不进图。 |
| LangGraph HITL interrupt/resume | **不采用为审批账本或写执行**。官方 interrupt 无限等待、节点重跑与 BR-082/085/040、WriteIdentity 分离冲突。审批 = HTTP 命令 + `ApprovalRequest`。 |

普通 GET 任务/审核查询不构成引入以上框架的理由。

---

## 6. 本专题开放问题

- 深链 URL path `[TBD-FE]`。
- SSE 是否一期交付（推荐：一期 GET 轮询即可，SSE 为增强）。
- confirm 校验失败：`pending` 保留 vs `rejected`（推荐 `rejected`）。
- 开发深链未命中是否统一 403 以防探测 `[TBD-BIZ]`。
- CSRF / cookie vs bearer `[TBD-BE]`。
