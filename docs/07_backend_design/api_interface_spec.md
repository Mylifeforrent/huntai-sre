# HuntAI SRE API 契约

- **Status**: Draft（Stage 7 API 契约；**非已冻结**）
- **日期**: 2026-09-15
- **文件名**: Stage 7 约定产出 `api_interface_spec.md`
- **输入（权威从高到低）**:
  1. [`../06_architecture_design/frontend_backend_boundary_spec-v1.0.md`](../06_architecture_design/frontend_backend_boundary_spec-v1.0.md)（协作原则与「必须调 API」的操作）
  2. [`../06_architecture_design/architecture_spec.md`](../06_architecture_design/architecture_spec.md)（用户入口若写 `docs/07_backend_design/architecture_spec.md`，以 Stage 6 路径为准；`AGENTS.md`：架构正文不进 07）。专题 [`../06_architecture_design/02_api_workflow_and_review.md`](../06_architecture_design/02_api_workflow_and_review.md) 锁定资源与动词语义。
  3. [`data_model_spec.md`](./data_model_spec.md)（表、分级、禁止出域字段）
- **技术底座**: FastAPI + Pydantic v2；REST JSON；前缀 `/api/v1/`（ADR-0008 / `tech_stack.md`）。
- **纪律**: API **只能**落实边界文档及其引用的命令/查询语义。禁止新增业务 API（无通用 Job、无审批中心、无无 `alert_id` 对话、无 silence/ack、无 kubeconfig 上传）。禁止把页面 L1 文案做成字段。无法确定的标 **[TBD-BE]** / **[TBD-BIZ]** / **[TBD-FE]** / **[TBD-INFRA]**，禁止假默认值。

**一句话**: 浏览器只谈本契约。权限、配额、Citation id、调查主状态只在控制面。一期进度以 GET 调查为权威（ADR-0007）。

---

## 1. 范围、非目标与闭集

### 1.1 范围

- 一期与二期 **HTTP** 端点（二期在一期调用须 fail-close，见 §1.3）。
- 每个端点：Method + Path、鉴权、Request/Response、错误码、幂等/分页/排序/筛选、限流档。
- 全局错误码表（唯一、分段、含语义）。
- FastAPI / Pydantic 分层建议（§10）。**本文不生成实现代码。**

### 1.2 非目标

- OpenAPI YAML 文件、实现、Alembic、编排。
- 前端路由 path（仍 **[TBD-FE]**）；深链 **查询参数** 必须是 `cluster_id` + `fingerprint`。
- SSE / WebSocket（ADR-0007：SSE 可选增强，**不能**替代 GET；一期客户端不得依赖）。
- 领域事件总线、worker 内部 RPC。
- `AutoCreateInvestigation`、`ProposeWriteAction`：系统/调查图经控制面，**无**对应 HTTP。

### 1.3 一期 vs 二期

| 标记 | 含义 |
|---|---|
| 一期 | 一期验收必须实现 |
| 二期 | 路径与错误码预留。一期客户端 **禁止**调用；若调用 → HTTP 404 或 `HNT-CAP-7001`（`capability_not_enabled`）。禁止返回「即将上线」 |

一期 PATCH 集群源时若请求体出现 `env` / `write_remediation` / `scale_max` / Loki / WriteIdentity 字段 → `HNT-CAP-7001`（边界规范 §4：一期设置页无二期字段）。

### 1.4 禁止出现的 API（永禁）

与权限矩阵禁止列、交互 0.3、Business Model §9 对齐。实现中出现即违规。

| 禁止 | 原因 |
|---|---|
| 取消调查、通用 job 取消 | A02；关页不 cancel |
| Alertmanager silence / ack；xMatters；出站 IM/邮件 | BR-044 |
| kubectl delete/apply/exec；读 secrets；Playwright | §9 |
| 上传 kubeconfig / 回显密钥明文 | BR-046、NFR-021、边界 §1 |
| RelatedLink 删除/编辑 | **[TBD-BIZ]**（IX-CNF-06）；未拍板不提供 |
| 用户自助 TeamBinding | BR-014 |
| 独立审批中心、无 `alert_id` 的 ChatSession | BR-089 |
| 开发触发扫描；namespace 子集扫描 | §4 |
| 全文搜索 alertname | 交互：**[TBD-BIZ]**，本期不做 |
| 匿名读任何业务资源 | NFR-020 |
| 把模型输出的 Citation id 写入 | BR-031 |

---

## 2. 协议约定

### 2.1 传输

| 项 | 约定 |
|---|---|
| Base | `/api/v1/` |
| 资源名 | 小写、连字符、名词复数（`project_rules.md` §1.3） |
| Body | `application/json`；UTF-8 |
| 时间 | ISO-8601，UTC，后缀 `Z` |
| 成功 | **直接**返回资源或列表信封；不包多余 `success: true` |
| 失败 | 统一 error envelope（`02` §1.3） |
| CORS | 生产默认禁止 `*` **[ASSUMPTION]**（安全专题 6.4）；同源托管（ADR-0008）后浏览器不跨域 |
| CSRF / Cookie vs Bearer | **[TBD-BE]** / **[TBD-FE]**。写命令必须带登录会话。未决前契约把凭证称为 **Session**，不冻结 Cookie 名或 `Authorization` 头 |

### 2.2 成功列表信封

```json
{
  "items": [],
  "next_cursor": null,
  "page_size": 50
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `items` | array | 当前页 |
| `next_cursor` | string \| null | 不透明游标；无下一页为 `null` |
| `page_size` | integer | 实际条数上限回显 |

无 `total` **[ASSUMPTION]**：避免对开发做全表 COUNT 泄露不可见总量。Empty vs No Result 由「是否带筛选」在前端区分，不靠服务端第三态。

### 2.3 失败信封

```json
{
  "error": {
    "code": "HNT-AUTH-1001",
    "message": "请先登录"
  }
}
```

- `error.code`：§4 全局唯一码。
- `error.message`：面向用户的短句；**禁止**栈、SQL、密钥、Prompt、未脱敏日志、内部表名。
- **禁止** `details`、`trace_id` 对前端暴露内部结构（`02` §1.3）。关联排障用服务端日志字段，不进本 envelope。
- HTTP 状态与业务码分开：同一 `HNT-AUTHZ-2001` 始终配 403；客户端以 `code` 映射页态，不以自造 HTTP 语义。

### 2.4 公共请求头

| Header | 何时 | 规则 |
|---|---|---|
| `Idempotency-Key` | 调查创建、扫描发起、二期 confirm/reject、break-glass 登录 | UUID 或 ≥16 字符不透明串 **[ASSUMPTION]**。前端禁用不能替代。同一键 + 同一用户 + 同一路径：重放返回首次成功响应。同一键不同 body → `HNT-CONFLICT-5102` |
| `Authorization` 或会话 Cookie | 除 OIDC 跳转、break-glass、ingest 外 | 形态 **[TBD-BE]** |
| Ingest 凭证 | 仅 API-080 | 算法 **[TBD-BE]**（D1）。契约只要求「与用户 Session 分离」 |

### 2.5 分页 / 排序 / 筛选（全局）

| 查询参数 | 适用 | 规则 |
|---|---|---|
| `page_size` | 列表 | 默认 50，最大 100 **[ASSUMPTION]**；超上限按 100 截断并仍 200，或 `HNT-VAL-3001`。推荐截断 |
| `cursor` | 列表 | 服务端签发；客户端原样回传。禁止客户端构造 UUID 当游标猜下一页 |
| 排序 | 列表 | **服务端固定**，客户端不可传 `sort=`（防改序绕过 ACL 窗口） |

固定排序：

| 列表 | 排序 |
|---|---|
| Alert | `last_webhook_at DESC`，同分 `id DESC` |
| Alert 下 Investigation | `created_at DESC` |
| ScanRun | `created_at DESC` |
| TeamBinding | `(user_id, label_key, label_value)` |
| User | `created_at ASC` |

筛选 **只允许** 各端点声明的 query。未知 query **忽略**（不 400），以免前端加实验参数阻断；**禁止**把未知 filter 当 SQL。不做全文搜索。

### 2.6 幂等（写命令）

| 操作 | 键 | 行为 |
|---|---|---|
| AM webhook | `cluster_id` + fingerprint | upsert；一期不建 Investigation |
| 人点调查 | `Idempotency-Key`（落 `investigations.idempotency_key`） | 连点一条；**新键**允许对同一 Alert 再开人工调查（占新的一次日额） |
| 扫描发起 | `Idempotency-Key` + 集群 running 互斥 | 重放返回同一 ScanRun；另一请求且已有 `running` → `HNT-CONFLICT-5101` |
| 二期 confirm/reject | `Idempotency-Key` + `approval_request_id` 行锁 | 只执行一次 |
| break-glass 登录 | `Idempotency-Key` 可选 | 有键则同一挑战只建一次会话 |

GET 可重试。无「取消调查」故无取消幂等。

---

## 3. 鉴权模型

### 3.1 主体

| 主体 | HTTP | 说明 |
|---|---|---|
| 匿名 | 仅 API-001/002/003 | 禁止预览业务数据（边界 §1） |
| 登录 User | Session | `role=sre` \| `developer`；`is_break_glass` 权限同 SRE，审计必须可区分 |
| IngestCredential | 仅 API-080 | 不得附带用户 Session 当作 ingest 身份 |
| `auto-investigator` | **无** HTTP | 不可登录、不可 confirm |

会话过期 → `HNT-AUTH-1002`，回 PG-001（EX-01.3）。

开发 TeamBinding 为空：**[TBD-BIZ]** 是否阻断登录。未决前 **[ASSUMPTION]** 允许登录，列表 Empty（EX-01.4）。

### 3.2 端点鉴权缩写

| 缩写 | 含义 |
|---|---|
| 匿名 | 无 Session |
| 登录 | 任一有效 User（含 break-glass） |
| SRE | `role=sre` 或 `is_break_glass=true` |
| 开发 | `role=developer` 且非 break-glass |
| Ingest | 仅 IngestCredential |
| 可见 Alert | 登录 **且** 通过 TeamBinding 行级可见性（SRE 含 `unscoped`；开发仅标签匹配） |
| 二期写确认 | 登录且过 R2：SRE 任意 `env`；开发仅 `non_production` 且目标 namespace ∈ 绑定 |

前端隐藏扫描/设置 **不是** 安全边界。开发直链设置/一期扫描 → `HNT-AUTHZ-2001`，不泄露资源正文。

### 3.3 行级可见性（所有 Alert / Investigation / ToolCall）

与 `02` §1.4、BR-017/022/055 同一函数：

1. 无行 → 深链 `HNT-RES-5001`（未入库）；调查保留删除 → `HNT-RES-5003`。
2. 有行但当前用户不可见 → `HNT-AUTHZ-2001`，**禁止**返回 `alertname`、labels、annotations（IX-PERM-02）。
3. 能见 Alert ⇒ 可只读其下全部 Investigation（含他人与 `trigger=auto`）。
4. 开发用 query 扩大范围：服务端过滤，列表不出现越权行（EX-04.4）。

开发深链「未命中」统一 403 以防探测仍 **[TBD-BIZ]**。未决前按 `02`：未入库 404、不可见 403。

---

## 4. 错误码（全局唯一）

分段：`HNT-{SEG}-{NNNN}`。`NNNN` 全局不复用。HTTP 为建议值，实现必须与本表一致。

### 4.1 AUTH `10xx` — 认证

| code | HTTP | 语义 | 页态 |
|---|---|---|---|
| `HNT-AUTH-1001` | 401 | 未登录访问受保护资源 | 回 PG-001，记回跳 |
| `HNT-AUTH-1002` | 401 | 会话过期 | EX-01.3 |
| `HNT-AUTH-1003` | 401 | break-glass 失败；**不区分**用户是否存在 | EX-02.1 |
| `HNT-AUTH-1004` | 401 | OIDC 未完成（拒绝/取消/超时） | EX-01.2 |
| `HNT-AUTH-1005` | 403 | CSRF 校验失败（策略落地后） | Error；无内部细节 |
| `HNT-AUTH-1006` | 429 | 登录类限流 | 稍后重试；不泄露账号 |

### 4.2 AUTHZ `20xx` — 授权

| code | HTTP | 语义 | 页态 |
|---|---|---|---|
| `HNT-AUTHZ-2001` | 403 | 角色或标签不可见；深链不可见；开发直链扫描/设置 | No Permission；不泄露 alertname |
| `HNT-AUTHZ-2002` | 403 | 开发触发扫描（一期整页、二期直调 POST） | EX-07.1 / EX-16.1 |
| `HNT-AUTHZ-2003` | 403 | 开发访问设置写/读（PG-009–012） | EX-09.1 等 |
| `HNT-AUTHZ-2004` | 403 | 二期 confirm 不满足 R2（production / namespace） | IX-PERM-09 |

`2002`/`2003`/`2004` 对前端可区分入口；对攻击者 message 仍用同一句「没有权限」，**不**说明「你是开发所以不能扫」。实现可将细分码留给客户端已登录壳，匿名探测一律 `2001`。**[ASSUMPTION]**：已登录用户返回细分码；匿名永远 `1001`。

### 4.3 VAL `30xx` — 校验

| code | HTTP | 语义 |
|---|---|---|
| `HNT-VAL-3001` | 422 | 通用字段校验（Pydantic） |
| `HNT-VAL-3002` | 400 | 深链缺 `cluster_id` 或 `fingerprint` |
| `HNT-VAL-3003` | 422 | RelatedLink 非绝对 URL / kind 非法 |
| `HNT-VAL-3004` | 409 | `cluster_id` 唯一冲突 |
| `HNT-VAL-3005` | 409 | 集群目录已达上限 5 |
| `HNT-VAL-3006` | 422 | 请求试图上传 kubeconfig 或文件字段 |
| `HNT-VAL-3007` | 422 | TeamBinding `label_key`/`label_value` 非法或空 |
| `HNT-VAL-3008` | 422 | 放行未指定用户；闸数字非正整数 |
| `HNT-VAL-3009` | 422 | 二期 `env` 非法；空提交策略见 IX-VAL-10 |
| `HNT-VAL-3010` | 422 | 二期 continue 试图离开本 Investigation / 无 `alert_id` |

### 4.4 QUOTA `40xx` — 配额与组织闸

| code | HTTP | 语义 | 占用日额 |
|---|---|---|---|
| `HNT-QUOTA-4001` | 409 | 当日人工新建已满 30（含放行窗口后仍满） | 否 |
| `HNT-QUOTA-4002` | 409 | 组织关闭新调查 | 否 |
| `HNT-QUOTA-4003` | 409 | 组织 token 闸未配置（不得视为无限） | 否 |
| `HNT-QUOTA-4004` | 422 | 放行额度/时限字段出现，但业务数字仍 **[TBD-BIZ]** 未批准 | 否 |

人点并发达 NFR-011 **不是**错误（BR-054）→ 201。

### 4.5 RES `50xx` — 资源

| code | HTTP | 语义 |
|---|---|---|
| `HNT-RES-5001` | 404 | 深链参数合法但 Alert 未入库 |
| `HNT-RES-5002` | 404 | 调查/扫描/绑定/集群 id 不存在（且不可见时走 403 而非本码） |
| `HNT-RES-5003` | 404 | 调查已按保留策略删除；message 固定语义「调查已按保留策略删除」；禁止伪造 RCA |
| `HNT-RES-5004` | 404 | ScanRun 不存在 |
| `HNT-RES-5005` | 404 | ClusterSource 不存在 |
| `HNT-RES-5006` | 404 | ToolCall / Citation / ApprovalRequest 不在该调查或不存在 |

### 4.6 CONFLICT `51xx`

| code | HTTP | 语义 |
|---|---|---|
| `HNT-CONFLICT-5101` | 409 | 同集群已有 `running` ScanRun（A03） |
| `HNT-CONFLICT-5102` | 409 | 同一 `Idempotency-Key` 配了不同 body |
| `HNT-CONFLICT-5103` | 409 | 审批非 `pending`（已 confirmed/rejected/void） |
| `HNT-CONFLICT-5104` | 409 | 二期 scale 超当前×2 / 超 `scale_max` / 当前 0 拉起 / 未配 `scale_max` |
| `HNT-CONFLICT-5105` | 409 | 二期调查已离开 `running`，pending 已/将 `void` |
| `HNT-CONFLICT-5106` | 409 | 二期 continue 时调查非 `running`（未决前 **[ASSUMPTION]**） |

### 4.7 INGEST `60xx`

| code | HTTP | 语义 |
|---|---|---|
| `HNT-INGEST-6001` | 401 | ingest 凭证缺失或错误；不回显 |
| `HNT-INGEST-6002` | 400 | 缺 `cluster_id`（或无法对应 ClusterSource） |
| `HNT-INGEST-6003` | 422 | payload 无法规范化；**不**留半行 |
| `HNT-INGEST-6004` | 429 | 该凭证 HTTP 限流 |
| `HNT-INGEST-6005` | 404 | `cluster_id` 未注册 |

### 4.8 CAP / SYS

| code | HTTP | 语义 |
|---|---|---|
| `HNT-CAP-7001` | 404 | 能力未开启（一期打二期字段/端点） |
| `HNT-SYS-9001` | 500 | 系统失败；无内部细节 |
| `HNT-SYS-9002` | 503 | 控制面只读依赖不可用且无法完成该写（少用；调查创建失败不占额） |

`completed` / `inconclusive` / `scan_failed` / `void` 是**资源状态**，不是错误码。

写身份不可用（EX-15.9）→ `HNT-SYS-9002` 或确认后 `HNT-CONFLICT-5103` 保持 pending：**推荐**一次判定 `rejected` + message「写身份不可用」（`02` §3），并写审计。精确二选一 **[TBD-BE]**；未决前 **[ASSUMPTION]** 转 `rejected`，避免重复 confirm。

---

## 5. 字段脱敏与禁止出现在 Response 的列

对照 `data_model_spec.md` §8。下列 **永远不得**出现在任何成功/失败 JSON、日志样例、OpenAPI example：

| 列 / 载荷 | 分级 | API 替代 |
|---|---|---|
| `users.password_hash` | 高敏 | 无 |
| `users.idp_subject` | 敏感 | 无；审计用内部 `user_id` |
| `users.email` | 敏感 | 持久化仍 **[TBD-BE]**；未决前 **不返回** |
| `ingest_credentials.secret_hash` / `secret_ciphertext` | 高敏 | `ingest_configured: bool` |
| `cluster_sources.prometheus_credential_ciphertext` | 高敏 | `prometheus_configured: bool` |
| `cluster_sources.kube_sa_token_ciphertext` | 高敏 | `readonly_sa_configured: bool` |
| `*_key_id`（KEK 版本） | 内部 | **不返回**（减少密钥管理面） |
| `write_identities.token_ciphertext` | 高敏 | `write_identity_configured: bool` |
| `loki_credentials.credential_ciphertext` | 高敏 | `loki_configured: bool` |
| `investigations.skill_snapshot` | 敏感 | 只回 `skill_loaded` / `skill_match_scope` |
| `investigations.alert_snapshot` / `visibility_snapshot` | 敏感 | 告警用 Alert DTO；绑定不回快照 |
| `investigations.token_usage` | 内部 | 不回（计量口径 **[TBD-BE]**） |
| ToolCall **未截断**原文 | — | 只回 `input_truncated` / `output_truncated` |
| `audit_events.payload` 全量 | 敏感 | 设置页只回最小化变更摘要 |
| Prompt 原文、LLM Key | L0 | 无 |

不可见 Alert：Response 不得含 `alertname`。列表对开发不得含 `unscoped=true` 行。

派生字段允许：`human_investigations_remaining_today`、`llm_available`、`awaiting_write_approval`（二期）、`configured` 布尔。剩余次数 **不**存库（数据模型 §10）。

---

## 6. API 清单

| ID | 阶段 | Method | Path | 鉴权 | 幂等 | 限流档 | 来源 |
|---|---|---|---|---|---|---|---|
| API-001 | 一期 | GET | `/api/v1/auth/oidc/start` | 匿名 | 否 | RL-LOGIN | TF-01；边界「跳转 OIDC」 |
| API-002 | 一期 | GET | `/api/v1/auth/oidc/callback` | 匿名（IdP 回跳） | 否 | RL-LOGIN | TF-01 |
| API-003 | 一期 | POST | `/api/v1/auth/break-glass` | 匿名 | Header 可选 | RL-LOGIN-BG | TF-02；边界「提交 break-glass」 |
| API-004 | 一期 | GET | `/api/v1/session` | 登录 | — | RL-READ | NAV-000 壳；配额/LLM 横幅 |
| API-010 | 一期 | GET | `/api/v1/alerts` | 登录 | — | RL-READ | TF-04；边界 Alert 可见性 |
| API-011 | 一期 | GET | `/api/v1/alerts/by-identity` | 登录 | — | RL-READ | TF-03 深链 |
| API-012 | 一期 | GET | `/api/v1/alerts/{alert_id}` | 可见 Alert | 看过副作用 | RL-READ | TF-04/05；BR-045 |
| API-013 | 一期 | GET | `/api/v1/alerts/{alert_id}/investigations` | 可见 Alert | — | RL-READ | PG-005 L3 |
| API-014 | 一期 | POST | `/api/v1/alerts/{alert_id}/related-links` | 可见 Alert | 否 | RL-WRITE | TF-08 |
| API-020 | 一期 | POST | `/api/v1/alerts/{alert_id}/investigations` | 可见 Alert | **必须** Header | RL-INV | TF-05；边界「创建调查」 |
| API-021 | 一期 | GET | `/api/v1/investigations/{investigation_id}` | 可见其 Alert | — | RL-POLL | TF-06；ADR-0007 |
| API-022 | 一期 | GET | `/api/v1/investigations/{investigation_id}/tool-calls/{tool_call_id}` | 同上 | — | RL-READ | 边界「打开已成功只读 ToolCall 副本」 |
| API-023 | 二期 | POST | `/api/v1/investigations/{investigation_id}/continuations` | 可见其 Alert | Header 建议 | RL-INV | TF-13「继续查」 |
| API-030 | 一期 | POST | `/api/v1/cluster-sources/{cluster_source_id}/scan-runs` | SRE | **必须** Header | RL-SCAN | TF-07；边界「扫描触发」 |
| API-031 | 一期 | GET | `/api/v1/scan-runs` | SRE；二期开发只读 | — | RL-READ | PG-007 |
| API-032 | 一期 | GET | `/api/v1/scan-runs/{scan_run_id}` | SRE；二期开发 namespace 相交 | — | RL-READ | PG-008 |
| API-040 | 一期 | GET | `/api/v1/cluster-sources` | 登录（字段按角色投影） | — | RL-READ | 列表筛集群；PG-009 |
| API-041 | 一期 | POST | `/api/v1/cluster-sources` | SRE | 否 | RL-SENS | TF-09 RegisterClusterSource |
| API-042 | 一期 | GET | `/api/v1/cluster-sources/{cluster_source_id}` | SRE | — | RL-READ | PG-009 |
| API-043 | 一期 | PATCH | `/api/v1/cluster-sources/{cluster_source_id}` | SRE | 否 | RL-SENS | TF-09 更新；二期 TF-17 同路径扩字段 |
| API-044 | 一期 | POST | `/api/v1/cluster-sources/{cluster_source_id}/ingest-credentials/rotate` | SRE | Header 建议 | RL-SENS | BR-016 轮换；不回显 |
| API-050 | 一期 | GET | `/api/v1/team-bindings` | SRE | — | RL-READ | TF-10 |
| API-051 | 一期 | POST | `/api/v1/team-bindings` | SRE | 否 | RL-SENS | UpdateTeamBinding |
| API-052 | 一期 | DELETE | `/api/v1/team-bindings/{team_binding_id}` | SRE | 否 | RL-SENS | 同上 |
| API-060 | 一期 | GET | `/api/v1/users` | SRE | — | RL-READ | TF-11 选用户 |
| API-061 | 一期 | POST | `/api/v1/users/{user_id}/quota-overrides` | SRE | Header 建议 | RL-SENS | GrantDailyQuotaOverride |
| API-070 | 一期 | GET | `/api/v1/organization` | 登录（字段按角色投影） | — | RL-READ | TF-12；Skill 映射读 |
| API-071 | 一期 | PATCH | `/api/v1/organization` | SRE | 否 | RL-SENS | SetOrgTokenGate；PG-010 Git URL |
| API-080 | 一期 | POST | `/api/v1/ingest/alertmanager` | Ingest | upsert 键 | RL-INGEST | UpsertAlert；边界外 AM |
| API-090 | 二期 | POST | `/api/v1/approval-requests/{approval_request_id}/confirm` | 二期写确认 | **必须** Header | RL-SENS | TF-15 |
| API-091 | 二期 | POST | `/api/v1/approval-requests/{approval_request_id}/reject` | 二期写确认 | **必须** Header | RL-SENS | TF-15 EX-15.7 |

路由注册：`/alerts/by-identity` 必须排在 `/{alert_id}` 之前。

无 GET Citation 独立资源：Citation 嵌在 API-021；点开证据走 API-022（ToolCall 副本）。Citation.id 仅展示，不可写。

---

## 7. 端点契约

公共：未列出的错误均可叠加 `HNT-SYS-9001`。写命令未登录 → `1001`/`1002`。

### 7.1 身份

#### API-001 `GET /api/v1/auth/oidc/start`

- **鉴权**: 匿名
- **Query**: `return_to` 可选，深链回跳；必须是站内相对路径 **[ASSUMPTION]**，禁止外链开放重定向
- **成功**: `302` 到 IdP。IdP 产品名 **[TBD-BIZ]**
- **错误**: `1004` 不在此发生（发生在 callback）；`1006`；非法 `return_to` → `3001`
- **限流**: RL-LOGIN

#### API-002 `GET /api/v1/auth/oidc/callback`

- **鉴权**: 匿名（持 IdP `code`/`state`）
- **成功**: `302` 到 `return_to` 或 PG-003；Set-Session
- **错误**: `1004` → 重定向登录页；禁止建会话
- **限流**: RL-LOGIN

#### API-003 `POST /api/v1/auth/break-glass`

**Request**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `username` | string | 是 | 本地标识；不在错误中回显是否存在 |
| `password` | string | 是 | 只用于比对；日志禁止记录 |

**Response `200`**

同 API-004 的 `SessionResponse`（含 `is_break_glass=true`）。

**错误**: `1003`（用户名空/错/口令错同一码）；`1006`；`3001`。

**限流**: RL-LOGIN-BG（严于 OIDC）。

#### API-004 `GET /api/v1/session`

**Response `200` `SessionResponse`**

| 字段 | 类型 | 分级 | 说明 |
|---|---|---|---|
| `user_id` | uuid | 内部 | |
| `display_name` | string \| null | 内部 | 持久化 **[TBD-BE]**；可空则前端用角色文案 |
| `role` | `sre` \| `developer` | 内部 | |
| `is_break_glass` | bool | 内部 | UI 横幅；权限同 SRE |
| `llm_available` | bool | 内部 | 运行时；关则 IX-FB-12 横幅，不隐藏告警 |
| `new_investigations_paused` | bool | 内部 | 组织开关投影 |
| `token_gate_configured` | bool | 内部 | `false` 时调查按钮须按 EX-05.3 处理 |
| `human_investigations_remaining_today` | int | 内部 | `30 - 当日成功 human`，加放行 **[TBD-BIZ]** 后的有效剩余；下限 0 |
| `quota_override_until` | datetime \| null | 内部 | 仅当当前用户有放行且未过期 |

**禁止**: `password_hash`、`idp_subject`、`email`、组织 token 已用量明细（口径 TBD）。

**错误**: `1001`/`1002`。

---

### 7.2 Alert 与 RelatedLink

#### API-010 `GET /api/v1/alerts`

- **鉴权**: 登录；服务端 ACL
- **Query**: `cluster_id` 可选；`status`=`firing`\|`resolved` 可选；`cursor`；`page_size`
- **禁止 query**: `unscoped`、`team`、`namespace` 作为「扩大可见性」参数。开发传了也不得看到未绑定行
- **排序**: §2.5
- **Response**: 列表信封，`items[]`=`AlertListItem`

`AlertListItem`

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | uuid | |
| `cluster_id` | string | |
| `fingerprint` | string | |
| `alertname` | string \| null | 可见行才有 |
| `status` | `firing` \| `resolved` | |
| `unscoped` | bool | 开发列表不应出现 `true` 行 |
| `labels_public` | object | **仅** `team`/`owner`/`namespace`/`severity`（若有）。其余 labels **[TBD-BIZ]** 脱敏清单未定前 **不**整包返回 **[ASSUMPTION]**：列表足够用这四键 |
| `starts_at` | datetime \| null | |
| `last_webhook_at` | datetime | |
| `firing_cycle` | int | 二期跳过句用；一期可回 |

**错误**: `1001`；`3001`（status 非法）。

#### API-011 `GET /api/v1/alerts/by-identity`

- **Query**: `cluster_id` **必须**；`fingerprint` **必须**
- **成功 `200`**: `AlertDetail`（同 API-012），并执行「看过」审计（与打开详情相同）
- **错误**: 缺参 `3002`；未入库 `5001`；不可见 `2001`（无 alertname）

未登录深链：前端先 API-001，`return_to` 带原查询串；本 API 仍 401。

#### API-012 `GET /api/v1/alerts/{alert_id}`

- **副作用**: 写 `audit_events.action=alert_seen`（BR-045）。无按钮、无 Toast。重复打开可再写 **[ASSUMPTION]**（审计允许重复看过）
- **Response `200` `AlertDetail`**: `AlertListItem` + `annotations_safe` + `related_links[]` + `generator_url` + 配额投影

| 额外字段 | 类型 | 说明 |
|---|---|---|
| `annotations_safe` | object | 清单未定前 **[ASSUMPTION]** 只回非密钥类短文本键；禁止整包原始 annotations 若可能含 token。fail-close：无把握的键丢弃 |
| `related_links` | `RelatedLinkResponse[]` | 只读列表 |
| `human_investigations_remaining_today` | int | 与 session 一致，供确认层 |
| `llm_available` | bool | |

`RelatedLinkResponse`: `id`, `kind` (`grafana`\|`loki`\|`kibana`\|`elastic`), `url`, `created_at`。无密钥。

**错误**: `1001`；不存在或不可见：存在但对当前用户不可见 → `2001`；UUID 无行 → `5002`（避免对开发用 404 枚举 **[TBD-BIZ]**；未决前不可见统一 `2001`，无行 `5002` 仅当调用者本可见范围应能知道「不是 ACL」——实现上对开发 **一律 `2001`** **[ASSUMPTION]**，对 SRE 无行 `5002`）。

#### API-013 `GET /api/v1/alerts/{alert_id}/investigations`

- **鉴权**: 可见该 Alert
- **Response**: 列表信封，`InvestigationSummary[]`（无 ToolCall 正文）

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | uuid | |
| `status` | `running` \| `completed` \| `inconclusive` | |
| `trigger` | `human` \| `auto` | 一期仅 `human`；二期才出现 `auto` |
| `created_at` | datetime | |
| `initiator_principal` | `user` \| `auto-investigator` | 不回发起人口令 |
| `citation_count` | int | 派生 |

一期客户端忽略 `auto`。二期 PG-005「打开自动调查」用本列表。

SRE 可见本 firing 因并发/闸跳过 auto 的提示：从审计最小化字段 `auto_investigate_skipped_reason` 可选挂在 `AlertDetail` **[ASSUMPTION]**：`null` \| `concurrency_cap` \| `org_gate`。开发不回该字段。

#### API-014 `POST /api/v1/alerts/{alert_id}/related-links`

**Request**

| 字段 | 类型 | 必填 |
|---|---|---|
| `kind` | enum | 是 |
| `url` | string | 是；绝对 URL |

host 允许列表 **[TBD-BIZ]**；未定前只做 URL 句法（EX-08.1）。

**Response `201`**: `RelatedLinkResponse`。

**错误**: `2001`；`3003`。无删除端点。

---

### 7.3 Investigation

#### API-020 `POST /api/v1/alerts/{alert_id}/investigations`

- **Header**: `Idempotency-Key` **必须**，否则 `3001`
- **Request body**: `{}` 或省略。禁止客户端传 `trigger`、`status`、Citation
- **校验顺序**: 可见性 → 组织闸配置且未暂停 → 日额 → 落库 `running` `trigger=human` → 入队 worker。失败不插行、不占日额
- **无 LLM / 禁出域**: 仍 `201`；资源 `llm_unavailable=true`；禁止把终态预写成 `completed`

**Response `201` `InvestigationCreated`**

| 字段 | 类型 |
|---|---|
| `id` | uuid |
| `status` | 恒 `running` |
| `alert_id` | uuid |
| `llm_unavailable` | bool |

前端必须用 `id` 进 PG-006（IX-FB-01）。

**错误**: `2001`；`4001`/`4002`/`4003`；`9001`/`9002`（均不占额）。人点并发 **不**错误。

**限流**: RL-INV（HTTP 层）+ 业务日额。

#### API-021 `GET /api/v1/investigations/{investigation_id}`

进度权威。建议 `Cache-Control: no-store`。轮询间隔 **[TBD-FE]**。

**Response `200` `InvestigationDetail`**

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | uuid | 亦 LangGraph `thread_id`，但对前端只是 id |
| `alert_id` | uuid | |
| `status` | enum | 禁止前端发明第四终态 |
| `terminal_reason` | `zero_citation` \| `cost_llm` \| `cost_tools` \| `cost_wall_clock` \| `no_llm` \| null | `completed` 必须 null |
| `trigger` | enum | |
| `initiator_principal` | enum | |
| `llm_call_count` | int | ≤8 |
| `readonly_tool_call_count` | int | ≤12 |
| `wall_clock_ms` | int | pending 人批期间不加 |
| `wall_clock_limit_ms` | int | 恒 150000（BR-040），非配置幻想 |
| `skill_loaded` | bool | false →「未加载组织约束」 |
| `skill_match_scope` | `global` \| `cluster` \| `team` \| null | |
| `llm_unavailable` | bool | |
| `readonly_loop_finished_at` | datetime \| null | |
| `awaiting_write_approval` | bool | 二期派生：`running` ∧ 只读循环已停 ∧ 存在 pending。一期恒 `false` |
| `claims` | `ClaimResponse[]` | |
| `tool_calls` | `ToolCallTimelineItem[]` | 含失败行 |
| `related_links` | 同 Alert | 标注非证据由前端；API 不把它们放进 `claims` |
| `write_actions` | `WriteActionResponse[]` | 一期 `[]`；二期人批区 |

`ClaimResponse`: `id`, `seq`, `statement`, `citation_ids[]`（控制面铸造的 UUID）。

`ToolCallTimelineItem`:

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | uuid | |
| `seq` | int | |
| `tool_name` | string | 白名单名 |
| `is_readonly` | bool | |
| `outcome` | `succeeded` \| `failed` | |
| `truncated` | bool | |
| `citation_id` | uuid \| null | 仅成功只读且已铸造 |
| `started_at` / `finished_at` | datetime \| null | |

时间线 **不**内嵌 `output_truncated`（防列表过大）。点 Citation / 展开走 API-022。

失败工具：`citation_id=null`，前端不可点成证据（IX-FB-05）。

`WriteActionResponse`（二期；一期不出现或空数组）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | uuid | |
| `verb` | `restart` \| `scale` | 禁止其他动词 |
| `target_kind` | string | |
| `target_namespace` | string | 敏感但有权可见 |
| `target_name` | string | |
| `current_replicas` / `desired_replicas` | int \| null | |
| `preview` | object | 缺 preview 不得出确认；内容按 BR-082 |
| `execution_result` | object \| null | **禁止**被当成 Citation |
| `approval` | `ApprovalResponse` \| null | |

`ApprovalResponse`: `id`, `status` (`pending`\|`confirmed`\|`rejected`\|`void`), `void_reason`, `decided_at`。**不**回 `confirmed_by` 的 IdP 标识；可回 `confirmed_by_user_id`。

**错误**: `2001`；`5003` 保留删除；`5002`。

#### API-022 `GET .../tool-calls/{tool_call_id}`

- 必须同属该调查。失败调用仍可 GET（时间线展开），但不得带 `citation_id`
- **Response**: `id`, `tool_name`, `outcome`, `truncated`, `is_readonly`, `input_truncated`, `output_truncated`, `citation_id`
- WriteAction 执行结果 **不是** ToolCall，本路径 404/`5006`

#### API-023 〔二期〕`POST .../continuations`

**Request**: `{ "message": string }` 绑定本调查；服务端忽略客户端 `alert_id`，以资源上的为准（IX-VAL-12）。

**成功 `202`**: 接受继续只读循环；仍受 8/12/150s。

**错误**: `7001` 一期；`3010`；`5106`；`2001`。

---

### 7.4 Scan

#### API-030 `POST /api/v1/cluster-sources/{cluster_source_id}/scan-runs`

- **鉴权**: SRE。开发 → `2002`
- **Header**: `Idempotency-Key` 必须
- **Body**: `{}`。禁止 `namespace` 过滤字段（EX-07.4）
- **成功 `201`**: `{ "id", "status": "running", "trigger": "manual", "cluster_source_id" }`
- **错误**: `5101` 已有 running；`5005`；`2002`；`2003` 不用于本路径（用 `2002`）

#### API-031 `GET /api/v1/scan-runs`

- **Query**: `cluster_source_id` 可选；分页
- **一期**: 仅 SRE。开发 → `2001`/`2002`
- **二期**: 开发可 GET，`trigger` 含 `scheduled`；Finding 在详情过滤
- **Item**: `id`, `cluster_source_id`, `cluster_id`, `trigger`, `status`, `created_at`。无 `error_summary` 堆栈；失败可回短 `error_summary`

#### API-032 `GET /api/v1/scan-runs/{scan_run_id}`

**Response**: ScanRun + `findings[]`

`FindingResponse`: `id`, `rule_id`, `namespace`, `resource_kind`, `resource_name`, `summary`。`payload` 仅含无密钥规则字段；无把握则不回 **[ASSUMPTION]**。

二期开发：只返回 `namespace` 与绑定相交的 Finding；无 namespace 命中对开发不可见（数据模型 **[ASSUMPTION]**）。`findings_visible_count` 按过滤后计数。无 ack/close 字段。

---

### 7.5 Catalog 与设置

角色投影：开发调 API-040 只得 `id, cluster_id, display_name`（列表筛选用）。SRE 得配置 DTO。开发调 API-042/041/043/044/050–071 → `2003`。

#### API-040 `GET /api/v1/cluster-sources`

SRE `ClusterSourceListItem`: `id`, `cluster_id`, `display_name`, `prometheus_configured`, `readonly_sa_configured`, `ingest_configured`。一期 **禁止** `env`/`write_remediation`/`scale_max`/`loki_configured`/`write_identity_configured`。

#### API-041 `POST /api/v1/cluster-sources`

**Request**（写密钥只进、不回）

| 字段 | 类型 | 必填 |
|---|---|---|
| `cluster_id` | string | 是 |
| `display_name` | string \| null | 否 |
| `prometheus_base_url` | string \| null | 否 |
| `prometheus_credential` | string \| null | 写-only |
| `kube_api_url` | string \| null | 否 |
| `kube_sa_token` | string \| null | 写-only |
| `ingest_secret` | string \| null | 写-only；静态 bearer 则哈希存储 |

禁止 `kubeconfig` 字段（出现 → `3006`）。超过 5 → `3005`。id 冲突 → `3004`。

**Response `201`**: 与 GET 详情相同的**无密钥** DTO。

#### API-042 / API-043 GET/PATCH 同一 DTO

PATCH：省略 = 不变；显式 `null` 是否清除凭证 **[TBD-BE]**。未决前 **禁止**用 null 清除，须走 rotate 或未来专用命令。

二期 PATCH 增：`env`, `write_remediation`, `scale_max`（null=禁 scale）, `loki_credential` 写-only, `write_identity_token` 写-only。开写修复但未配写身份：保存可 `200`，`write_identity_configured=false`（EX-17.4）。

#### API-044 `POST .../ingest-credentials/rotate`

**Request**: `{ "secret": string, "label": string \| null }` 客户端生成秘密。**禁止**响应回显 `secret`。成功 `{ "ingest_configured": true, "rotated_at": datetime }`。

---

#### API-050 `GET /api/v1/team-bindings`

**Query**: `user_id` 可选。

**Item**: `id`, `user_id`, `label_key`, `label_value`, `created_at`。`label_value` 敏感、仅 SRE。

可附 `recent_changes[]` 最多 5 条最小化：`occurred_at`, `actor_user_id`, `action=team_binding_changed`。**禁止**完整审计台、禁止 payload 密钥。

#### API-051 `POST /api/v1/team-bindings`

**Request**: `{ "user_id", "label_key": "team"|"owner"|"namespace", "label_value" }`

唯一冲突 → `3001`/`3007`。写 AuditEvent。禁止自助：调用者必须是 SRE。

#### API-052 `DELETE /api/v1/team-bindings/{team_binding_id}`

`204` 无 body。写审计。无撤销栈。

---

#### API-060 `GET /api/v1/users`

SRE 选放行对象。**Item**: `id`, `display_name`, `role`, `is_break_glass`, `quota_override_until`。禁止 email / idp_subject / password_hash。

分页适用。无「创建用户」API（OIDC 即时供应 **[ASSUMPTION]**；break-glass 部署保证一行）。

#### API-061 `POST /api/v1/users/{user_id}/quota-overrides`

**Request**: 指定用户在 path。Body 的 `extra_count` / `until` **[TBD-BIZ]**。未批准数字前：

- 若 body 为空 `{}` → `4004`（禁止服务端填 +10/24h）
- 维护人批准后本契约再扩字段（须 change_log）

成功暂不可用不等于「无限放行」。

---

#### API-070 `GET /api/v1/organization`

| 字段 | 开发可见 | SRE 可见 |
|---|---|---|
| `name` | 是 | 是 |
| `new_investigations_paused` | 是 | 是 |
| `token_gate_configured` | 是 | 是 |
| `daily_token_limit` | 否 | 是；未配置 `null`（不得显示无限） |
| `skill_git_url` | 否 | 是；可空 |
| `llm_available` | 是 | 是 |

#### API-071 `PATCH /api/v1/organization`

SRE。字段可组合：`new_investigations_paused`（关闸须前端二次确认，API 仍一次 PATCH；未确认不发请求）、`token_gate_configured`、`daily_token_limit`（正整数或 `null` 表示未配置）、`skill_git_url`（可空；非空须 Git URL 句法 IX-VAL-06）。

关闸或闸未配置：阻断新调查与二期 auto（BR-042）。本 API 不提供「只关 auto 仍开人工」开关。

---

### 7.6 Ingest

#### API-080 `POST /api/v1/ingest/alertmanager`

- **鉴权**: IngestCredential。带用户 Session **不能**替代（BR-016）
- **凭证放置** **[TBD-BE]**: Header bearer 或 HMAC。契约要求失败统一 `6001`，不区分「无此集群密钥」
- **Body**: Alertmanager webhook JSON。必须能解析出 `cluster_id`（标签或 query **[TBD-BE]** 与 ClusterSource 对应）。`fingerprint` 优先 AM 自带，否则按 BR-020 计算
- **一期**: 只 upsert Alert，**不**创建 Investigation
- **二期**: 同事务后半段条件 AutoCreate；失败跳过不得回滚 Alert（BR-072）。无额外 HTTP
- **成功**: `202` `{ "status": "upserted" }`。不回 labels/告警正文给 AM
- **错误**: `6001`–`6005`；`6004` 限流

规范化失败不留半行。重复 firing upsert 幂等。

---

### 7.7 二期人批

#### API-090 `POST /api/v1/approval-requests/{id}/confirm`

- 一期：`7001`
- **Header**: `Idempotency-Key` 必须
- **Body**: `{}`。禁止改 preview
- **校验**: 登录人 ≠ `auto-investigator`；R2；调查仍 `running`；`write_remediation` 且 WriteIdentity；scale 规则 BR-084
- **成功 `200`**: 更新后的 `WriteActionResponse`（`approval.status=confirmed`）
- **错误**: `2004`；`5103`/`5104`/`5105`；写身份失败见 §4.8

#### API-091 `POST .../reject`

- 不执行写。`200` `{ "status": "rejected" }`
- 已终态 → `5103`。幂等重放返回首次结果

---

## 8. 限流策略

业务闸（日额 30、组织 token、auto 并发 5、扫描互斥）**不是** HTTP 限流；冲突用 409 业务码。HTTP 限流用 **429** + `HNT-AUTH-1006` 或 `HNT-INGEST-6004`。数字除已拍板业务值外均为 **[ASSUMPTION]**，禁止写成 SLA。

| 档 | 作用对象 | 建议阈值 **[ASSUMPTION]** | 超限 |
|---|---|---|---|
| **RL-LOGIN-BG** | API-003；按 IP | 5 次 / 5 分钟 | `1006`；不泄露账号 |
| **RL-LOGIN** | API-001/002；按 IP | 30 次 / 分钟 | `1006` |
| **RL-INGEST** | API-080；按凭证 | 峰值 **[TBD-INFRA]**；未定前每凭证 120 次 / 分钟 | `6004`；upsert 仍须在放行后幂等 |
| **RL-INV** | API-020、API-023 | 每用户 10 次 / 分钟（防连点打满 worker）；日额仍 409 | 429 不占日额 |
| **RL-SCAN** | API-030 | 每用户 10 次 / 分钟 | 429；`5101` 优先于 429 若已 running |
| **RL-SENS** | API-041/043/044/051/052/061/071/090/091 | 每用户 20 次 / 分钟 | 429；保护密钥轮换与放行 |
| **RL-POLL** | API-021 | 每用户 60 次 / 分钟 | 429；前端须遵守 **[TBD-FE]** 间隔 |
| **RL-READ** | 其余 GET | 每用户 120 次 / 分钟 | 429 |
| 全局 IP | 全 API | **[TBD-INFRA]** | 未定前可只做上表 |

`Retry-After` 秒数 **[TBD-BE]**。限流计数器存哪（内存 / PostgreSQL）**[TBD-INFRA]**；一期单实例内存可接受，不宣称多副本一致。

登录与敏感操作 **必须**落地 RL-LOGIN-BG、RL-LOGIN、RL-SENS、RL-INGEST。缺任一不得进入 Stage 11 合入（对照本表）。

---

## 9. 页态 / 任务流 / 命令追溯

| 必须调 API 的操作（边界 + `02` + 命令表） | API |
|---|---|
| 跳转 OIDC | API-001/002 |
| 提交 break-glass | API-003 |
| 壳：角色、横幅、剩余次数、LLM 关 | API-004（及调查确认层 API-012） |
| 深链 `cluster_id`+fingerprint | API-011 |
| 告警列表 + 筛选 | API-010；筛选项集群 API-040 |
| 打开详情 + 看过 | API-012 |
| 详情调查列表 | API-013 |
| POST 创建调查，返回 id | API-020 |
| GET 轮询进度 | API-021 |
| 打开只读 ToolCall 副本 | API-022 |
| 保存 RelatedLink | API-014 |
| SRE 分析该集群 | API-030–032 |
| 注册/改集群；密钥不回显 | API-041–044 |
| TeamBinding | API-050–052 |
| 临时放行 | API-060/061 |
| 组织闸 / Skill Git | API-070/071 |
| AM webhook | API-080 |
| 〔二期〕继续查 | API-023 |
| 〔二期〕confirm/reject | API-090/091 |
| 〔二期〕打开 auto 调查 | API-013 + API-021（无创建 API） |
| 〔二期〕Loki citation | 无独立 API（工具结果在 021/022） |
| 〔二期〕开发只读 Finding | API-031/032 ACL |
| 〔二期〕env / 写修复 / Loki / WriteIdentity | API-043 扩字段 |

命令无 HTTP：`AutoCreateInvestigation`、`ProposeWriteAction`。

---

## 10. FastAPI / Pydantic 设计建议（不实现）

Stage 11 落地时建议分层，避免 ORM 模型直接当 Response（会把密文列漏出）。

### 10.1 模块

与领域模块对齐的 router 前缀：`auth`、`alerts`、`investigations`、`scan_runs`、`cluster_sources`、`team_bindings`、`users`、`organization`、`ingest`、`approval_requests`（二期）。依赖注入：`CurrentUser`、`require_sre`、`alert_visibility`。

### 10.2 模型命名（PascalCase）

| 建议类名 | 用途 |
|---|---|
| `ErrorEnvelope` / `ErrorBody` | 全局 exception handler |
| `SessionResponse` | API-004 |
| `BreakGlassLoginRequest` | extra=forbid；密码 `SecretStr` |
| `AlertListItem` / `AlertDetail` | 无 annotations 整包 |
| `RelatedLinkCreateRequest` / `RelatedLinkResponse` | |
| `InvestigationCreated` / `InvestigationDetail` / `InvestigationSummary` | |
| `ToolCallEvidenceResponse` | API-022；与 timeline item 分开 |
| `ScanRunCreateResponse` / `ScanRunDetail` / `FindingResponse` | |
| `ClusterSourceWriteRequest` | 密钥字段 `SecretStr`；`model_dump` 永不进日志 |
| `ClusterSourcePublicResponse` | 仅 `*_configured` |
| `TeamBindingCreateRequest` | |
| `OrganizationPatchRequest` | |
| `AlertmanagerIngestBody` | 按 AM 官方 webhook 形状校验；多余字段丢弃 |
| `ApprovalConfirmResponse` | 二期 |

共用：`ConfigDict(extra="forbid")`；UUID `uuid.UUID`；时间 `datetime` timezone-aware。写请求与读响应用 **不同类**。ORM 列 `password_hash` 不得出现在任何 `*Response` 的 `model_fields`。

### 10.3 FastAPI 行为建议

- 校验失败映射 `HNT-VAL-3001`，不要把 Pydantic `ctx` 里的输入口令返回。
- `HTTPException` 与领域异常分开；领域抛 `QuotaExceeded` → 409 + `4001`。
- OpenAPI 由 FastAPI 生成后须人工对照本文；example **禁止**真实密钥。
- 幂等中间件在进入 service 前按 `(user_id, path, key)` 查表。
- API-021 声明为安全 GET，但实现必须读库最新投影（禁止长时间缓存）。
- 不要为 SSE 加一期 router。

### 10.4 依赖与配置

会话、IdP、KEK、限流阈值只从仓库根 `.env` 读（`pydantic-settings`）。本文不写键值。新增键须同步 `.env.example` 且经批准。

---

## 11. 开放问题（仅 API）

| ID | 问题 | 标记 |
|---|---|---|
| A1 | Cookie vs Bearer、CSRF、Cookie 属性 | [TBD-BE]/[TBD-FE] |
| A2 | 开发深链 404 vs 403 探测 | [TBD-BIZ] |
| A3 | ingest 认证算法与 `cluster_id` 出现位置 | [TBD-BE] |
| A4 | 放行数字/时限；API-061 才能从 `4004` 升级为可执行 | [TBD-BIZ] |
| A5 | RelatedLink host 允许列表与删除 | [TBD-BIZ] |
| A6 | 轮询间隔、`Retry-After` | [TBD-FE]/[TBD-BE] |
| A7 | webhook HTTP 限流精确数字 | [TBD-INFRA] |
| A8 | 列表是否回 labels 整包 | 清单 [TBD-BIZ]；本文 fail-close 子集 |
| A9 | confirm 失败 `rejected` vs 保持 `pending` | [TBD-BE]；本文假设 rejected |
| A10 | 开发空绑定是否登录 | [TBD-BIZ] |
| A11 | PATCH 清除集群密钥的语义 | [TBD-BE] |
| A12 | 登出 HTTP | 交互无 TF；未列入清单，禁止发明直至产品要求 |

---

## 12. 完成前自检

- [x] 边界文档每个「必须调 API」的操作有对应 API-xxx（§9 表；含 OIDC、break-glass、深链、列表、详情/看过、创建调查、GET 进度、ToolCall 副本、扫描、设置、webhook、二期人批/继续查）
- [x] 每个端点鉴权与权限矩阵一致（§3、§6：开发禁扫描触发与设置；SRE/break-glass 同权可审计区分；ingest 分离；auto 无 HTTP；confirm 走 R2）
- [x] 错误码全局唯一（§4 分段 10xx–90xx 无复用）；Response 对照 §5 无高敏列、不可见不泄露 alertname
- [x] 未新增边界外业务 API（无 Job、无审批中心、无删除 RelatedLink、无 kubeconfig、无 SSE 一期依赖）
- [x] 无实现代码；Pydantic 仅为命名与分层建议（§10）
- [x] 架构路径按 Stage 6，未把架构正文写入 `docs/07_backend_design/`
