# 安全、可靠性与运行

- **Status**: Draft
- **日期**: 2026-09-14
- **角色**: Agent C（安全、可靠性与运行架构师）
- **权威输入**: Business Model §4、§6.7–6.8、§7；交互 IX-PERM / IX-FB-03
- **非目标**: 部署拓扑定稿（Stage 12）；编造可用性百分比；实现代码。

---

## 1. 身份认证

| 主体 | 机制 | 规则 |
|---|---|---|
| 人（日常） | 公司 IdP **OIDC** | 禁止匿名（BR-010、NFR-020）。IdP 产品名 `[TBD-BIZ]`，适配器可插拔 |
| 人（紧急） | 恰好一个 break-glass 本地账号 | 不走 OIDC；不用于日常（BR-011）；失败响应不区分用户是否存在（IX-VAL-02） |
| Alertmanager | IngestCredential | 与用户会话分离（BR-016）；密钥不回显 |
| 系统自动开查 | `auto-investigator` | 不可登录 UI（BR-073） |
| 集群只读 | 每集群 ServiceAccount | get/list/watch 工作负载与 events、读 pod logs；禁 secrets/exec/写（BR-047） |
| 集群写（二期） | WriteIdentity | 与只读 SA 分离；失败禁止 fallback（BR-086） |
| Loki（二期） | LokiCredential | 只读；无凭证则不调 Loki（BR-077） |
| LLM | BYOK / 网关 | 出域前脱敏；禁出域开关与网关地址 `[TBD-BIZ]`；无清单不得出域（NFR-025） |

会话过期后拒绝后续写，回登录（EX-01.3）。部署实例 **0** 份个人 kubeconfig（NFR-021）。本机开发约定（BR-046）不是运行时配置项。

OIDC 组 → TeamBinding 由管理员维护，禁止用户自助勾选（BR-014）。

---

## 2. 授权、组织、租户

- **租户**: 恰好一个 Organization（BR-002、NFR-016）。不实现多组织切换、部门级多租户（§9.33–34）。
- **授权主键**: 告警标签 `team` / `owner` / `namespace` 与 TeamBinding 匹配（BR-013）。无三者 → `unscoped`，仅 SRE/break-glass 可见（BR-022）。
- **行级过滤**: 列表与 GET 在服务端同一套可见性函数。禁止把全量 Alert 交给前端过滤。
- **开发 Loki**: 系统注入 namespace；去掉约束则拒绝（BR-076）。无 namespace 绑定则禁止 Loki。
- **开发 Finding（二期）**: namespace 相交过滤在服务端（BR-079）。
- **前端隐藏**: 一期开发看不到扫描/设置导航（IX-NAV-01）；直链必须 403。

不设 NOC、经理操作者。不与 xMatters 组绑定作为权限主键（§9.23）。

---

## 3. 证据「文件」安全、数据分级、隐私、保留

### 3.1 分级 `[ASSUMPTION]`（无独立合规分级文档）

| 级 | 例子 | 处理 |
|---|---|---|
| L0 密钥 | IngestCredential、SA token、WriteIdentity、LLM key | 仅密文或 secret 引用；禁止进调查库/日志/Citation（BR-052） |
| L1 可能含 PII/机密的遥测 | pod logs、Prom 查询原文、告警标签值 | 界面可对有权用户展示；**出域前可逆变脱敏**（BR-049）；Skill 可禁某 namespace log 外发 |
| L2 调查产物 | ToolCall 截断副本、Claim、Citation | 随 Investigation 90 天（BR-060） |
| L3 审计 | AuditEvent | 365 天（BR-061）；载荷脱敏 |
| L4 告警副本 | Alert | 90 天（BR-062） |

无用户上传通道，故无「上传杀毒/配额」设计。证据写入只来自白名单工具。

### 3.2 脱敏与出域

- 未覆盖字段必须有公开清单；清单未定 **不得出域**（BR-049、NFR-025）。
- 运行时默认可逆变脱敏后 BYOK 出域；架构预留禁止出域（§7.3）。默认值 `[TBD-BIZ]`。
- Citation 指向截断后的存储副本，禁止把未截断超大原文整包送入 LLM（BR-043）。截断字节 `[TBD-BIZ]`。

### 3.3 保留与删除

调度作业按 BR-060–062 删除到期行（调查树、Alert、审计）。删除完成时限 `[TBD-BIZ]`（NFR-033）。打开已删调查 URL → 空/错误「调查已按保留策略删除」（EX-06.8）。删除本身写审计（删除审计行到期后一并删）。

备份恢复见 §5：备份窗口不得把已到期数据无限复活为在线查询集——策略 `[TBD-INFRA]`。

---

## 4. 性能、容量、可用性

已拍板数字直接用；未拍板标 TBD，**禁止**「高可用」「能抗风暴」。

| ID | 指标 | 架构含义 |
|---|---|---|
| NFR-001–003 | 150s / 8 LLM / 12 工具 | worker 硬闸；pending 审批不计入 150s（A06） |
| NFR-004–006 | 入库/开查 P95 | `[TBD-BIZ]`；不对外宣称 SLA |
| NFR-010 | 20 人，结构按 1000 账号 | 单组织；TeamBinding 表按千人设计即可 |
| NFR-011 | 人点 running 一期按 5 规划 / 结构 50 | **不作为**创建硬闸（BR-054） |
| NFR-012 | 每用户每天 30 | 按自然日计数；时区 **[ASSUMPTION]** `Asia/Shanghai`（BR-041）；拒绝不占次数 |
| NFR-013 | 2–5 集群 | catalog 上限 5 |
| NFR-014 | webhook 峰值 | `[TBD-BIZ]`；ingest 须幂等，无数字前不做容量承诺 |
| NFR-015 | 组织日 token | `[TBD-BIZ]`；未配置 fail-close 新调查（BR-042） |
| NFR-017 | auto running ≤5 | 与人点分开计数 |
| NFR-018 | 每集群 60 分钟扫描 | 调度器；可与调查 worker 共享进程池但隔离并发槽 `[ASSUMPTION]` |

可用性百分比收口未涉及：**不写 SLA**。一期可接受单实例 + PostgreSQL；多副本与故障转移 `[TBD-INFRA]`（Stage 12）。

---

## 5. 备份、恢复、限流、监控、日志、告警

| 主题 | 一期策略 | 缺口 |
|---|---|---|
| 备份 | PostgreSQL 常规备份 `[TBD-INFRA]`（频率/RPO 未收口） | 不得把密钥明文打进备份日志 |
| 恢复 | 文档化 restore 演练在 Stage 12；本阶段只要求调查/审批/审计同库可恢复 | SQLite 不承担此职责（A05） |
| 限流 | 每用户调查日额 + 组织 token 闸 + 人点/auto 并发；webhook 按凭证限流数字 `[TBD-INFRA]` | 无公网 SaaS 滥用模型，但仍需防 AM 重放风暴 |
| 监控 | 进程健康、worker 积压、闸触发次数、webhook 4xx/5xx、调查终态计数 | 指标后端 `[TBD-INFRA]` |
| 日志 | 结构化；禁止 L0 与未脱敏 L1 | IX-FB-03 |
| 错误上报 | `.env.example` `SENTRY_DSN` 留空则不上报 | 不得把 Alert 正文/日志原文打进 Sentry extra |
| 产品出站告警 | **不做**（BR-044） | `ALERT_WEBHOOK_URL` 不接入产品通道（C4） |

LangGraph 生产 checkpointer 若启用必须落 PostgreSQL，避免进程重启丢失 `running` 图状态（官方 persistence）。即便图状态丢失，控制面仍能凭墙钟把调查标 `inconclusive`，不得标 `completed`。

---

## 6. 外部系统、部署边界、成本与运维风险

### 6.1 外部系统

| 系统 | 方向 | 失败模式 |
|---|---|---|
| IdP | 登录 | 失败不建会话（EX-01.2） |
| Alertmanager | 入 | 凭证/缺 cluster_id 拒绝；不回调改 AM |
| Prometheus | HuntAI→Prom 只读 | 工具失败，无 citation |
| kube-apiserver | 只读；二期写身份 | 禁止 fallback kubeconfig |
| Loki | 二期只读 | 无凭证则无工具行 |
| xMatters | 无 | 深链由运营改模板 |
| LLM 网关 | 出域可选 | G3：关模型时列表+扫描仍可用 |
| Git（Skill） | 调查开始拉取 | 空仓库允许；提示未加载约束 |
| Sentry | 可选出 | 见上 |

### 6.2 部署边界

- 配置唯一入口：仓库根 `.env`（`AGENTS.md`）。代码不硬编码。
- 已部署实例使用每集群只读 SA，禁止个人 kubeconfig（MVP-14）。
- Stage 12 才允许顶层 `docker-compose.yml` / `.github/`。本阶段不写编排文件。
- 编排形态已冻结（ADR-0008）：compose 三服务 api / worker / postgres；一期 **不上** Kubernetes。Playwright E2E 不进一期（产品永不做 Playwright；CI E2E 留 Stage 12 再议）。

### 6.3 成本

- 单次调查硬闸（OneUptime 同构数字，已拍板 BR-040）。
- 组织日 token 闸必须存在开关；数字未填 = 未配置 = 新调查 fail-close（BR-042）。
- 自动开查不计入用户 30 次，但仍计入组织 token 与 8/12/150s（BR-073–074）。关闸或闸未配置时 **阻断 auto**（BR-042）。

### 6.4 运维风险

| 风险 | 缓解 |
|---|---|
| 模型乱写集群 | 一期无写工具；二期策略写死 + 人批 + 分离写身份 |
| allowlist 被模型绕过 | 工具注册表在代码中静态；不按 MCP 发现加载（竞品 Holmes #2228） |
| 出域合规 | 无清单不出域；预留禁出域 |
| LLM 账单与 on-call 同时爆 | 8/12/150s + 日额 + 组织闸 + auto 并发 5 |
| 单人误关闸 | IX-CNF-04 确认 |
| 调研 SuperBizAgent 裸奔 | 默认鉴权、无 CORS `*` 作为生产默认 `[ASSUMPTION]` 待 Stage 11 落实 |

---

## 7. 缺失的非功能需求及其影响

| 缺失 | 影响 |
|---|---|
| 可用性 / RPO / RTO | 架构只承诺单组织可恢复备份意图，不承诺 99.x% |
| webhook 与开查 P95 | ingest 与 worker 不做无依据的容量调优承诺 |
| 自然日时区 | 配额日界不确定 |
| 脱敏字段清单 | 阻塞出域 |
| 截断阈值 | 阻塞大 logs 策略 |
| IdP 与会话存储（cookie 时长、刷新） | 阻塞安全基线细节 |
| webhook 认证算法（header token vs HMAC） | ingest 实现待 Stage 7 |
| 对象存储 | 大证据是否出库 |
| 多副本 worker 与队列 | 一期可进程内队列；多实例时需 PG 锁或队列 `[TBD-INFRA]` |

以上全部保持 `[TBD-*]`，fail-close：无清单不出域；无 token 数字则关新调查；无 `scale_max` 则禁 scale。
