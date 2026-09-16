# HuntAI SRE 产品需求文档（PRD）

- **Status**: Draft（Stage 8 综合交付；**非**已冻结。业务规则权威仍是 Business Model，本文不得改写 BR）
- **日期**: 2026-09-15
- **产品**: HuntAI SRE（公司内单一 Organization 的可信 AI 运维层）
- **输入（权威从高到低；本会话正文引用 ≤ 5 份）**:
  1. [`docs/03_problem_modeling/business_model.md`](../03_problem_modeling/business_model.md)
  2. [`docs/04_interaction_design/interaction_design_summary.md`](../04_interaction_design/interaction_design_summary.md)
  3. [`docs/05_prototype/prototype_review.md`](../05_prototype/prototype_review.md)（**文件不存在**，见 §0）
  4. [`docs/06_architecture_design/frontend_backend_boundary_spec-v1.0.md`](../06_architecture_design/frontend_backend_boundary_spec-v1.0.md)
  5. [`docs/07_backend_design/api_interface_spec.md`](../07_backend_design/api_interface_spec.md)
- **页面编号核对（路径引用，本会话未把该文当权威正文）**: [`docs/06_architecture_design/frontend_design_spec-v1.0.md`](../06_architecture_design/frontend_design_spec-v1.0.md)（**文件不存在**，见 §0）
- **纪律**: 综合已拍板结论。禁止重新定义业务、禁止把 `[TBD-*]` / `[ASSUMPTION]` 写成已批准数字、禁止把二期能力写入一期验收、禁止新增 Gate 1 未冻结的页面或能力。上游冲突标 `[CONFLICT]`，不自行裁决。

**一句话（不改写）**: Web 是唯一产品入口。被呼叫的人用深链或告警列表打开同一条 Alertmanager firing 告警，完成可区分的 cited RCA 或 `inconclusive`。一期仅人工点「调查」；二期在 `severity=critical` 时可自动开查，调查内可 Citation Loki LogQL，写修复仅人批 `restart`/`scale`，并展示 60 分钟零 token 扫描的 Finding。

---

## 0. 输入缺口与 `[CONFLICT]`

本文 **不裁决** 下列冲突。未裁决前：页面闭集 = 交互摘要已编号的 **PG-001–PG-012**；一期任务流 = **TF-01–TF-12**；二期任务流 = **TF-13–TF-17**（独立验收）。禁止新增 PG / 导航项 / 能力入口。

| ID | 冲突 | 未裁决前本文用法 |
|---|---|---|
| C-PRD-01 | `AGENTS.md`：缺约定输出禁止进入下一阶段。Stage 5 约定的 `prototype_link.md`、`prototype_review.md` **均不存在**。`docs/06_architecture_design/architecture_spec.md` 写明原型缺失「不阻塞、不补造」。 | 不补造 Gate 1 记录；不从 `docs/05_prototype/huntai-sre.zip` 发明页面。能力闭集以 Business Model §8 / §11 与交互 0.1 / 0.2 为准。 |
| C-PRD-02 | Stage 6 约定产出 `frontend_design_spec-v1.0.md` 不存在（边界规范：等 Stage 7 后再写）。无法按该文核对页面编号。 | PG-xxx 以交互摘要 §1.2 为准。 |
| C-PRD-03 | 交互 PG-010 展示「全局 / 集群 / team」映射范围；API-070/071 仅组织级 `skill_git_url`。 | 不新增 Skill 映射 API，不删除 PG-010。实现前须维护人裁决「单仓库内路径约定」或「补契约」。 |
| C-PRD-04 | BR-041 要求 SRE 可临时放行日额度；API-061 在额度/时限数字仍 `[TBD-BIZ]` 时，空 body → `HNT-QUOTA-4004`，禁止服务端填「+10 次 / 24h」。 | 一期 AC 验收「拒绝假默认」；不把「放行成功写入具体次数」标为一期 Pass。 |
| C-PRD-05 | 开发深链未命中：404 PG-004 vs 统一 403 以防探测，仍 `[TBD-BIZ]`。交互未定前：无记录 → PG-004，不可见 → No Permission。 | AC 按该未定前规则写；不得把「统一 403」写成已拍板。 |
| C-PRD-06 | 交互 EX-07.3：同一集群已有 `running` ScanRun 则禁用再分析为 `[ASSUMPTION]`；API-030 已有 `running` → `HNT-CONFLICT-5101`。 | AC 按 API 契约（409 + 5101）与 PG-007 Success-Failure / 按钮禁用验收；不把「允许并行 ScanRun」写成需求。 |
| C-PRD-07 | Stage 7 `gate2_review.md` 约定产出不存在；API 契约 Status=Draft。 | 接口以 `api_interface_spec.md` 现稿为准；不把 API 当已冻结。 |

`[TBD-BIZ]` / `[TBD-FE]` / `[TBD-BE]` / `[TBD-INFRA]` / `[ASSUMPTION]` 全集见交互摘要 §7 与 API 契约 §11，本文不填假默认。

---

## 1. 需求背景与用户目标

### 1.1 背景（不改写）

HuntAI SRE 接在现网 Kubernetes / Prometheus / Alertmanager / xMatters **之上**：Alertmanager 告警入库后由人点「调查」（二期在 `severity=critical` 时可自动开查），用只读工具产出**服务端 citation** 的 RCA。xMatters 继续负责呼叫。无模型时仍能把 firing 告警和规则扫描结果当作 finding。

调查权威在 HuntAI；呼叫 / 升级权威在 xMatters。HuntAI 不是全栈可观测、不是 Prom/AM 替代、不是对外 SaaS。

### 1.2 用户与角色

仅三种人 + 一种系统身份（Business Model §4）。不设 NOC、经理操作者。

| 角色 | 一期目标 | 二期增量（独立验收） |
|---|---|---|
| SRE / 平台 | 看全部 Alert（含 `unscoped`）；点调查；整集群零 LLM 扫描；配置数据源 / Skill / TeamBinding / 额度闸 | 任意 `env` 人批 restart/scale；看跳过自动开查原因；配 `env` / 写修复 / Loki / WriteIdentity |
| 开发 on-call | 只看 TeamBinding 匹配的 Alert；可对可见 Alert 调查；导航只有「告警」 | 导航增加只读「扫描」；LogQL 强制 namespace；仅 `non_production` 且 namespace ∈ 绑定可 confirm |
| break-glass 本地管理员 | 恰好一个；权限同 SRE；常驻「非日常调查」横幅；不走 OIDC | 同 SRE |
| IngestCredential | 只 upsert Alert；无 UI | 同左；满足 BR-070 时由控制面自动开查（无 HTTP） |
| `auto-investigator` | 无 | 不可登录、不可 confirm、不占用户日额 |

### 1.3 用户目标（G1–G6）

判定抄自 Business Model §1，不新增 KPI 数字。

| ID | 目标 | 可观察判定 | 一期 / 二期 |
|---|---|---|---|
| G1 | 被 xMatters 叫起的人能打开**同一条** firing 告警并完成 cited 调查 | 深链或列表定位 `cluster_id` + fingerprint；终态为「有 citation 的完成」或 `inconclusive`，界面可区分 | 一期必须 |
| G2 | 开发只能调查自己团队标签下的告警；SRE 能看全部（含 `unscoped`） | 权限矩阵；无标签告警对开发不可见 | 一期必须 |
| G3 | 无 LLM Key / 禁出域时仍展示 firing，且 SRE 能跑按需规则扫描 | 关模型后列表与扫描仍可用；扫描路径零 LLM | 一期必须 |
| G4 | 不成为监控权威、呼叫权威；一期不执行任何外部写 | 无 Prom/AM/xMatters 替换；一期无 silence / ack / page / IM / 邮件出站 / kubectl 写 | 一期必须 |
| G5 | 一期 20 人可用，结构能撑 1000 注册账号 | NFR-010、NFR-011（人点并发是规划容量，不是创建硬闸） | 一期必须 |
| G6 | 不推翻 G1–G5：critical 自动开查、LogQL citation、人批 restart/scale、定时规则扫描 | Business Model §11；BR-070 起 | **二期独立验收** |

**禁止当目标**: 公开 MTTR / RCA 准确率数字；独立对话产品 / 无 `alert_id` 的 Ask。

运营依赖（不是 HuntAI 功能项，但是 G1 完整体验需要）：xMatters 模板加入深链。模板未改时人走告警列表。产品不提供「去改 xMatters」按钮。

### 1.4 验收分期（不得串期）

| 批次 | 范围 | 不得用对方充数 |
|---|---|---|
| 一期 | Business Model §8 共 **20** 项；交互 TF-01–TF-12；API 标「一期」的端点 | 不得用 §11 / TF-13–TF-17 / API-023/090/091 充一期 Pass |
| 二期 | Business Model §11 共 **8** 项；TF-13–TF-17 | 一期未交付不得开始用二期项充数 |

---

## 2. 功能清单（关联 BR-xxx）

页面仅 PG-001–PG-012。无标注 = 一期必须；标 **〔二期〕** 的不进入一期验收。

### 2.1 一期功能（映射 MVP 20 项）

| ID | 功能 | 页面 | 主 BR | 协作 API |
|---|---|---|---|---|
| FN-01 | 公司 IdP OIDC 登录；禁止匿名 | PG-001 | BR-010、BR-012、NFR-020 | API-001、API-002、API-004 |
| FN-02 | 恰好一个 break-glass 本地管理员（非日常主入口） | PG-002 | BR-011 | API-003、API-004 |
| FN-03 | IdP 组 → `team` / `owner` / `namespace` 的 TeamBinding；管理员可改；禁止自助勾选 | PG-011 | BR-013、BR-014、BR-015 | API-050、API-051、API-052 |
| FN-04 | 注册 2–5 个 ClusterSource；每集群 AM 双发（xMatters receiver 保留）由运营完成，产品不改 AM | PG-009 | BR-004、BR-016、BR-023、BR-024 | API-040–API-044 |
| FN-05 | IngestCredential 校验；按 BR-020 fingerprint；upsert Alert | 无登录 UI | BR-016、BR-020、BR-021、BR-023 | API-080 |
| FN-06 | 告警列表：firing / resolved；SRE 全部；开发仅匹配；`unscoped` 仅 SRE | PG-003 | BR-017、BR-022、BR-026 | API-010、API-040 |
| FN-07 | 人工点「调查」创建 Investigation；入库不自动开调查 | PG-005 → PG-006 | BR-030、BR-041、BR-042、BR-054 | API-020、API-012 |
| FN-08 | 只读工具：kube get/describe/events/logs；Prom firing/pending 列表；即时 PromQL | PG-006 | BR-033、BR-035（一期禁 Loki/Elastic 查询） | API-021、API-022 |
| FN-09 | 服务端 Citation 与 Claim 绑定；零 Citation 必须 `inconclusive` 且界面可区分 | PG-006 | BR-031、BR-032 | API-021、API-022 |
| FN-10 | 成本闸：8 LLM / 12 工具 / 150s / 每用户每天 30 次人工调查；超限停并 `inconclusive` | PG-006、PG-012 | BR-040、BR-041、BR-042 | API-020、API-021、API-070、API-071、API-061 |
| FN-11 | 调查开始拉匹配 `SKILL.md`；无 Skill 提示「未加载组织约束」 | PG-006、PG-010 | BR-038、BR-039 | API-021、API-070、API-071 |
| FN-12 | 关闭 LLM 时 firing Alert 仍可展示 | PG-003、PG-005 | BR-026、BR-050 | API-004、API-010、API-012 |
| FN-13 | SRE 按需、零 LLM 的整集群规则扫描 | PG-007、PG-008 | BR-027、BR-028 | API-030、API-031、API-032 |
| FN-14 | 出域前可逆变脱敏；清单未定不得出域 | PG-006（无清单则无 LLM 完成态） | BR-049、BR-053 | API-020、API-021 |
| FN-15 | 已部署实例每集群只读 SA + Prom 只读；禁止上传 kubeconfig | PG-009 | BR-046、BR-047、BR-048 | API-041、API-043 |
| FN-16 | 稳定深链 `cluster_id` + fingerprint；未入库文案锁定 | PG-004、PG-005 | BR-051 | API-011 |
| FN-17 | 打开告警详情写本地「看过」AuditEvent；不同步 AM/xMatters | PG-005 | BR-045、BR-044 | API-012 |
| FN-18 | RelatedLink 保存 Grafana / Loki / Kibana / Elastic **URL**（非 Citation） | PG-005、PG-006 | BR-034、BR-037 | API-014 |
| FN-19 | AuditEvent：登录、绑定、开调查、扫描、数据源、额度放行、看过 | 设置页最小化摘要，无独立审计台 | BR-015、BR-041、BR-045 | 写路径随各命令；PG-011 可读最近变更 |
| FN-20 | 按 BR-060–BR-062 保留与到期删除；Web 为唯一产品入口 | PG-006 Empty（保留删除后） | BR-001、BR-002、BR-060–062 | API-021 `HNT-RES-5003` |

### 2.2 〔二期〕功能（映射 §11 共 8 项）

| ID | 功能 | 页面 | 主 BR | 协作 API |
|---|---|---|---|---|
| FN-21 | 仅 `severity` 精确等于 `critical` 时自动开查；缺键 fail-close；并发 ≤5 跳过并审计 | PG-005、PG-006 | BR-070–BR-074、BR-042 | API-080（无创建 HTTP）；API-013、API-021 |
| FN-22 | Loki LogQL 只读进调查白名单，可 Citation；禁止 Elastic 查询 API 与扒页 | PG-006 | BR-075、BR-077、BR-035 | API-021、API-022（无独立 LogQL 控制台） |
| FN-23 | 开发 LogQL 强制注入 TeamBinding.`namespace`；无 namespace 则禁止 | PG-006 | BR-076 | API-021、API-022 |
| FN-24 | 每集群每 60 分钟零 LLM 扫描；开发不可触发 | PG-007、PG-008 | BR-078 | API-031、API-032（无开发 POST） |
| FN-25 | 开发只读 Finding：namespace 相交；禁止用 UI 隐藏冒充自己触发 | PG-007、PG-008 | BR-079 | API-031、API-032 |
| FN-26 | 人批 `rollout restart`（Deployment / StatefulSet） | PG-006 人批区 | BR-080–BR-083、BR-085–BR-088 | API-090、API-091 |
| FN-27 | 人批 `scale`（Deployment；×2 与 `scale_max`）；未配 `scale_max` 禁止 scale | PG-006 人批区 | BR-084、NFR-019 | API-090 |
| FN-28 | `env` / `write_remediation` opt-in；WriteIdentity 与只读 SA 分离；ApprovalRequest preview→confirm | PG-009、PG-006 | BR-081、BR-082、BR-086、BR-087、BR-090 | API-043、API-021、API-090 |

二期另约束：调查页内「继续查」仍绑定原 Alert（BR-089）→ API-023。结束后是否仍可继续 `[TBD-BIZ]`；未定前 `[ASSUMPTION]` 仅 `running` 可继续。

### 2.3 明确不做的产品能力（功能清单负向）

见 §7。一期界面相对二期的禁用（交互 0.3）：不展示自动开查徽章、LogQL 工具条、ApprovalRequest、定时 ScanRun、`env` / `write_remediation` / `scale_max` / Loki / WriteIdentity 配置项。一期 PG-005 **禁止** Restart / Scale 按钮。

---

## 3. 核心流程与关键交互

流程权威：Business Model §2；页面变化权威：交互摘要 §2。此处只做综合索引，不改步骤。

### 3.1 一期主路径

| TF | 名称 | 用户操作摘要 | 成功落点 | 关键交互规则 |
|---|---|---|---|---|
| TF-01 | 公司账号登录 | 「使用公司账号登录」→ IdP | PG-003 或深链回跳 | IX-VAL-01；禁止匿名 |
| TF-02 | Break-glass | PG-001 次要链 → 用户名密码 | PG-003 + 非日常横幅 | IX-DUP-01；错误不区分账号是否存在 |
| TF-03 | 呼叫深链 | `cluster_id` + fingerprint | 命中可见 → PG-005 | 未登录先 TF-01/02；产品不改 xMatters 模板 |
| TF-04 | 列表定位 | 筛集群、firing/resolved | PG-005 | 列表不放「调查」；关 LLM 加横幅 |
| TF-05 | 人工调查 | PG-005 确认层后创建 | PG-006（以跳转为准，不以 Toast 代替，IX-FB-01） | IX-CNF-01；Idempotency-Key；拒绝不占日额 |
| TF-06 | 调查至可区分终态 | 停留或稍后打开；点 Citation | `completed` 或 `inconclusive` | 一期 GET 轮询（边界 §2）；关页不 cancel |
| TF-07 | 按需扫描 | SRE「分析该集群」 | PG-008 | IX-CNF-02；零 LLM；无 namespace 选择器 |
| TF-08 | RelatedLink | PG-005 添加绝对 URL | 列表只读 | 不进 Citation；无删除 API |
| TF-09 | 配置数据源 | 注册 AM/Prom/分析器 | PG-009 Default | 密钥不回显；无 kubeconfig 上传 |
| TF-10 | TeamBinding | 改映射并确认 | PG-011 | IX-CNF-05；无自助入口 |
| TF-11 | 临时放行 | 指定用户 | PG-012 | 数字未批准则不得假成功（C-PRD-04） |
| TF-12 | 组织 token 闸 | 关闭新调查 / 闸未配置 fail-close | PG-012 | 不得画成「无限」；关闸须确认 |

### 3.2 〔二期〕主路径

| TF | 名称 | 成功落点 | 关键约束 |
|---|---|---|---|
| TF-13 | 打开自动开查结果 | PG-006，`trigger=auto` | 无创建 HTTP；不改发起人；不占用户 30 次 |
| TF-14 | Loki 结果作 Citation | PG-006 证据层 | 无独立 LogQL 页；RelatedLink 的 Loki **界面 URL** 仍不是 Citation |
| TF-15 | 人批 restart/scale | 人批区 confirmed / rejected / void | preview 后**第二次**点击 confirm；写结果不可点成 Citation |
| TF-16 | 只读定时 Finding | PG-007/008 | 开发无「分析该集群」 |
| TF-17 | 配 env / 写修复 / Loki / WriteIdentity | PG-009 | 一期 PATCH 出现这些字段 → `HNT-CAP-7001` |

### 3.3 无 HTTP 的系统流程（仍须验收）

| 流程 | 说明 | 对用户可见 |
|---|---|---|
| Business Model §2.1 入库 | API-080 upsert；一期不建 Investigation | 失败表现为深链未命中或列表仍无该行，不伪造 Alert |
| §2.5 自动开查 | 控制面 `AutoCreateInvestigation` | PG-005 列表出现 `auto`；跳过则 SRE 见原因句，无「补开自动」 |
| §2.6 提议写修复 | 调查图 `ProposeWriteAction` | 仅 PG-006 人批区 |
| §2.7 定时扫描 | 每 ClusterSource 每 60 分钟 | PG-007 历史 `trigger=scheduled` |

### 3.4 导航（IX-NAV，一期）

- 未登录只允许 PG-001、PG-002。
- 已登录默认落地「告警」。一期开发主导航 **只有「告警」**；扫描与设置 **隐藏**（不是置灰）。
- SRE / break-glass：告警 / 扫描 / 设置。break-glass 常驻非日常横幅。
- 无「我的团队」自助切换器；无多 Organization 切换；无审批中心；无对话首页 / Ask。
- 深链 path `[TBD-FE]`；查询参数必须是 `cluster_id` + `fingerprint`。

〔二期〕开发主导航为「告警 + 扫描」（无触发按钮）；设置仍隐藏。

---

## 4. 状态与异常、权限

状态名与「每页是否适用」必须与交互摘要 **§4.2 总表** 一致。前端页态必须能由后端资源状态 + HTTP 推出（边界规范 §3）。禁止前端发明第四种调查终态。

### 4.1 状态定义（与交互 §4.1 逐字对齐）

| 状态 | 含义 |
|---|---|
| Default | 有权限、主数据已到、非编辑、非提交 |
| Loading | 首屏或刷新等待 |
| Empty | 合法空：该作用域从未有对象 |
| No Result | 作用域有数据，当前筛选为 0 |
| Error | 系统/网络失败 |
| No Permission | 角色或 Alert/namespace/`env` 范围不足 |
| Editing | 表单编辑中 |
| Submitting | 已提交等待写结果 |
| Success-Failure | 写操作结束后的业务成功或业务失败（含 `completed` / `inconclusive` / `scan_failed` / 额度拒绝 / `confirmed`/`rejected`/`void`）。不是系统 Error |

### 4.2 页面 × 状态矩阵（与交互 §4.2 一致）

| 页面 | Default | Loading | Empty | No Result | Error | No Permission | Editing | Submitting | Success-Failure |
|---|---|---|---|---|---|---|---|---|---|
| PG-001 | 适用：登录主屏 | 适用：探测会话 | 不适用 | 不适用 | 适用：OIDC 失败返回 | 不适用：本页即未授权入口 | 不适用 | 适用：等待 IdP | 适用：取消/失败登录 |
| PG-002 | 适用 | 适用 | 不适用 | 不适用 | 适用 | 不适用 | 适用：填账号密码 | 适用 | 适用：凭证错误 |
| PG-003 | 适用 | 适用 | 适用：可见 0 条 | 适用：筛选 0 条 | 适用 | 适用：会话角色异常（少见） | 不适用 | 不适用 | 不适用；LLM 关闭叠横幅于 Default |
| PG-004 | 适用：主文案即未入库 | 适用：查找中 | 不适用：与 Default 同义 | 不适用 | 适用：查询失败或参数残缺 | 适用：鉴权失败不应落到本页 | 不适用 | 不适用 | 不适用 |
| PG-005 | 适用 | 适用 | 不适用：无 Alert 走 PG-004 | 不适用 | 适用 | 适用：不可见 | 适用：RelatedLink | 适用：保存链接或创建调查 | 适用：额度/闸拒绝；链接保存成功。〔二期〕跳过自动开查用 L3 叠 Default |
| PG-006 | 适用：running 时间线或终态已渲染 | 适用 | 适用：保留期满删除 | 不适用 | 适用 | 适用 | 不适用改 Claim。〔二期〕继续查输入算轻量 Editing | 不适用创建（在 PG-005）。〔二期〕确认写修复算 Submitting | 适用：completed 与 inconclusive 必须用本态区分。〔二期〕confirmed/rejected/void |
| PG-007 | 适用 | 适用 | 适用：无 ScanRun；〔二期〕开发无相交 Finding | 不适用（一期）。〔二期〕若增加筛选再启用 | 适用 | 适用：一期非 SRE 整页；二期开发触发时 | 适用：SRE 选集群 | 适用：发起扫描 | 适用：发起失败 |
| PG-008 | 适用：completed 且有可见 Finding | 适用 | 适用：completed 但可见 Finding=0 | 不适用 | 适用 | 适用 | 不适用 | 不适用 | 适用：`scan_failed` |
| PG-009 | 适用 | 适用 | 适用：0 个集群 | 不适用 | 适用 | 适用 | 适用 | 适用 | 适用 |
| PG-010 | 适用 | 适用 | 适用：空仓库/无映射（允许） | 不适用 | 适用 | 适用 | 适用 | 适用 | 适用 |
| PG-011 | 适用 | 适用 | 适用：尚无映射 | 不适用 | 适用 | 适用 | 适用 | 适用 | 适用 |
| PG-012 | 适用：闸未配置而非无限 | 适用 | 不适用 | 不适用 | 适用 | 适用 | 适用 | 适用 | 适用 |

**PG-004 vs No Permission**: 可见性失败 → No Permission，不引用 alertname。确定无记录 → PG-004「尚未收到 AM webhook」。开发未命中是否统一 No Permission 见 C-PRD-05。

**PG-003 Empty vs No Result**: 无筛选仍 0 条 = Empty；用户改筛选后 0 条 = No Result。

**PG-006**: `running` 用 Default + Loading 片段，不用 Success-Failure。`completed` 与 `inconclusive` 必须在徽章、主标题、主内容三处可区分（IX-FB-04）。`inconclusive` 主标题禁止「根因是…」。

**〔二期〕PG-006**: 工具/LLM 已停且有 pending ApprovalRequest 时，主状态仍 `running`，L1 固定为「只读调查已结束，等待确认写操作」，禁止「正在分析 / 正在调查」（BR-090、IX-FB-11）。

### 4.3 业务状态机（不改写）

权威：Business Model §5。API 资源状态不是错误码。

| 对象 | 状态 | 对页面 |
|---|---|---|
| Alert | `firing` / `resolved`；`unscoped` 是标签不是状态 | PG-003 / PG-005 |
| Investigation | `running` → `completed` 或 `inconclusive`；鉴权失败不落 `running` | PG-006；〔二期〕有 pending 时保持 `running` |
| ScanRun | `running` → `completed` 或 `scan_failed` | PG-007 / PG-008 |
| ApprovalRequest〔二期〕 | `pending` / `confirmed` / `rejected` / `void` | PG-006 人批区 Success-Failure |

### 4.4 异常 → 页态 → 错误码

| 异常 | 页态 | 主错误码 |
|---|---|---|
| EX-01.1 未登录访问受保护 URL | 回 PG-001 Default（记回跳） | `HNT-AUTH-1001` |
| EX-01.2 IdP 拒绝/取消/超时 | PG-001 Error「登录未完成」 | `HNT-AUTH-1004` |
| EX-01.3 会话过期 | PG-001 Error「会话已过期」 | `HNT-AUTH-1002` |
| EX-01.4 开发 TeamBinding 空 | PG-003 Empty `[ASSUMPTION]` 允许登录 | — |
| EX-02.1 凭证错误 | PG-002 Error / Success-Failure | `HNT-AUTH-1003`（不区分用户是否存在） |
| EX-03.1 尚未入库 | PG-004 Default | `HNT-RES-5001` |
| EX-03.2 已入库不可见 | 当前页 No Permission | `HNT-AUTHZ-2001`（无 alertname） |
| EX-03.3 缺参数 | PG-004 Error「链接不完整」 | `HNT-VAL-3002` |
| EX-04.1 可见 0 条 | PG-003 Empty | 200 空列表 |
| EX-04.2 筛选 0 条 | PG-003 No Result | 200 空列表 |
| EX-04.3 加载失败 | PG-003 Error | `HNT-SYS-9001` |
| EX-04.4 开发扩大 query | 列表不出现越权行 | 服务端过滤 |
| EX-05.1 不可见调查 | PG-005 No Permission | `HNT-AUTHZ-2001` |
| EX-05.2 日额满 30 | PG-005 Success-Failure「今日调查次数已用尽」 | `HNT-QUOTA-4001`（不占额） |
| EX-05.3 关闸或闸未配置 | PG-005 Success-Failure | `HNT-QUOTA-4002` / `4003`（不占额） |
| EX-05.4 人点 running 达规划值 5 | **不拒绝**，进 PG-006 Default | 201（BR-054） |
| EX-05.5 双击确认 | 只一条调查；Submitting | 同一 `Idempotency-Key` |
| EX-05.6 无 LLM | PG-006 Default + 横幅；终态 inconclusive | 201 + `llm_unavailable=true` |
| EX-05.7 创建失败 | PG-005 Error 或 Success-Failure | `9001`/`9002`（不占额） |
| EX-06.1 Citation=0 | PG-006 Success-Failure（inconclusive） | 资源状态，非错误码 |
| EX-06.2 成本闸 | 同上（terminal_reason=cost_*） | 资源状态 |
| EX-06.3 工具失败 | 时间线失败行，不可点 Citation | `citation_id=null` |
| EX-06.4 无 Skill | Default 显示「未加载组织约束」 | `skill_loaded=false` |
| EX-06.5 关页 | 再打开见当前状态；无取消 | 无取消 API |
| EX-07.1 开发扫描 | 一期整页 No Permission | `HNT-AUTHZ-2002` |
| EX-07.2 集群不可达 | PG-008 Success-Failure | 资源 `scan_failed` |
| EX-07.3 已有 running | PG-007 Success-Failure / 按钮禁用 | `HNT-CONFLICT-5101` |
| EX-08.1 非 URL | PG-005 Editing 内联错误 | `HNT-VAL-3003` |
| EX-09.1 开发访问设置 | No Permission | `HNT-AUTHZ-2003` |
| EX-09.3 第 6 个集群 | 「注册集群」禁用或保存失败 | `HNT-VAL-3005` |
| EX-09.4 上传 kubeconfig | **不提供**控件；直调失败 | `HNT-VAL-3006` |
| EX-11.3 放行数字未批 | 不得出现「+10/24h」 | `HNT-QUOTA-4004` |
| EX-12.1 闸未配置画成无限 | **禁止该状态** | PG-012 Default 必须显示未配置 |

系统 Error 文案禁止栈、SQL、密钥、Prompt、未脱敏日志（IX-FB-03）。

### 4.5 权限表现（IX-PERM；前端隐藏不是安全边界）

| ID | 情境 | UI | 后端 |
|---|---|---|---|
| IX-PERM-01 | 一期开发：扫描、设置 | **隐藏** | 直链 403 |
| IX-PERM-01b | 〔二期〕开发：设置隐藏；扫描只读 | 扫描可见、触发隐藏 | POST 扫描 403 |
| IX-PERM-02 | 开发不可见 Alert | 列表不出现；深链 No Permission | 2001，无 alertname |
| IX-PERM-03 | 开发可见 Alert 的「调查」 | 主按钮可用 | API-020 |
| IX-PERM-04 | 关闸或日额用尽 | 「调查」禁用+原因 | 4001–4003 |
| IX-PERM-05 | `unscoped` | 仅 SRE/break-glass 列表可见 | 列表 ACL |
| IX-PERM-06 | 直链越权 | 页级 No Permission | 2001/2002/2003 |
| IX-PERM-07 | break-glass | 导航同 SRE + 横幅 | `is_break_glass=true` |
| IX-PERM-08 | silence 等永禁能力 | **不渲染** | 无对应 API |
| IX-PERM-09 | 〔二期〕开发 confirm | 仅 non_production 且 namespace ∈ 绑定 | `HNT-AUTHZ-2004` |
| IX-PERM-10 | 〔二期〕开发 Loki | 无去约束开关 | 拒绝执行 |
| IX-PERM-11 | 〔二期〕写工具默认关 | 整块不出现 | 无 pending 卡片 |
| IX-PERM-12 | 他人 / auto 调查 | 能见该 Alert 则可只读打开 | BR-055 |

角色能力表权威：Business Model §4，本文不重画「允许/禁止」列。

---

## 5. 数据与接口关联（API-xxx）

契约权威：[`api_interface_spec.md`](../07_backend_design/api_interface_spec.md)。数据表权威路径：[`docs/07_backend_design/data_model_spec.md`](../07_backend_design/data_model_spec.md)（本会话不展开字段）。前端不得铸造 Citation id、不得本地把 `inconclusive` 画成根因、不得靠改 query 看 unscoped。

### 5.1 操作 → API

| 必须调 API 的操作 | API | 页 |
|---|---|---|
| 跳转 OIDC | API-001、API-002 | PG-001 |
| 提交 break-glass | API-003 | PG-002 |
| 壳：角色、横幅、剩余次数、LLM 关 | API-004 | NAV-000；确认层兼 API-012 |
| 深链 | API-011 | PG-004 / PG-005 |
| 告警列表 + 筛选 | API-010；集群筛选项 API-040 | PG-003 |
| 打开详情 + 看过 | API-012 | PG-005 Default |
| 详情调查列表 | API-013 | PG-005 L3 |
| 创建调查 | API-020（`Idempotency-Key` 必须） | PG-005 Submitting → PG-006 |
| 轮询进度 | API-021 | PG-006 |
| 打开只读 ToolCall 副本 | API-022 | PG-006 证据层 |
| 保存 RelatedLink | API-014 | PG-005 Editing |
| SRE 分析该集群 | API-030–032 | PG-007 / PG-008 |
| 注册/改集群；密钥不回显 | API-041–044 | PG-009 |
| TeamBinding | API-050–052 | PG-011 |
| 用户列表（放行对象） | API-060 | PG-012 |
| 临时放行 | API-061 | PG-012 |
| 组织闸 / Skill Git | API-070、API-071 | PG-012、PG-010 |
| AM webhook | API-080 | 无用户页 |
| 〔二期〕继续查 | API-023 | PG-006 |
| 〔二期〕confirm / reject | API-090、API-091 | PG-006 人批区 |
| 〔二期〕打开 auto 调查 | API-013 + API-021（无创建 API） | PG-005 / PG-006 |
| 〔二期〕Loki citation | 无独立 API（嵌在 021/022） | PG-006 |
| 〔二期〕开发只读 Finding | API-031/032 ACL | PG-007 / PG-008 |
| 〔二期〕env / 写修复 / Loki / WriteIdentity | API-043 扩字段 | PG-009 |

无 GET Citation 独立资源。无取消调查 API。无审批中心 API。无 kubeconfig 上传 API。一期客户端禁止调用二期端点；若调用 → HTTP 404 或 `HNT-CAP-7001`。

### 5.2 一期客户端禁止出现的响应字段（摘录）

`env`、`write_remediation`、`scale_max`、`loki_configured`、`write_identity_configured` 不得出现在一期 ClusterSource DTO。`write_actions` 一期为空数组。`awaiting_write_approval` 一期恒 `false`。密钥、token、未脱敏原文、`password_hash`、`idp_subject`、email 不得出现在任何 Response。

### 5.3 错误码分段（验收时按 code 映射页态，不自造 HTTP 语义）

AUTH `10xx`、AUTHZ `20xx`、VAL `30xx`、QUOTA `40xx`、RES `50xx`、CONFLICT `51xx`、INGEST `60xx`、CAP `7001`、SYS `90xx`。完整表见 API 契约 §4。

---

## 6. 验收标准（AC-xxx）

规则：

1. 每条 AC 可独立 **Pass / Fail**；禁止「体验良好」「响应迅速」「高可用」等不可判定表述。
2. 每条 BR 至少被一条 AC 覆盖（§6.5 矩阵）。
3. 涉及 **UI 可见结果** 的 AC 必须同时标注 **PG-xxx** 与 **状态名**（Default / Loading / Empty / No Result / Error / No Permission / Editing / Submitting / Success-Failure），供 Stage 13 `highfi_page_map.md` 逐条覆盖。
4. 一期 AC 与〔二期〕AC 分列。一期回归不得把二期入口的「出现」标为 Pass。
5. NFR 中已拍板的数字写入判定；`[TBD-*]` 只验收「未填不得假装已满足 / 不得假默认」。

### 6.1 一期 — 身份与壳

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-001 | 无有效会话访问 PG-003–PG-012 任一业务 URL 时，浏览器落到 PG-001，且页面不展示任何 Alert / Investigation 正文。 | BR-010、NFR-020 | API-010 等返回 `HNT-AUTH-1001` | PG-001 **Default** |
| AC-002 | PG-001 Default 的 L1 主按钮文案为「使用公司账号登录」；点击后进入等待 IdP，且登录页不出现未定 IdP 产品商标。 | BR-010 | API-001 | PG-001 **Default** → **Submitting** |
| AC-003 | IdP 拒绝、取消或超时后回到登录页，会话未建立；可见文案为「登录未完成」。 | BR-010 | API-002 → `HNT-AUTH-1004` | PG-001 **Error** |
| AC-004 | 会话过期后下一次受保护请求回到登录页；可见文案为「会话已过期」。 | BR-010 | `HNT-AUTH-1002` | PG-001 **Error** |
| AC-005 | 匿名调用除 API-001/002/003 外的业务 GET/POST 均非 2xx 成功体。 | BR-010、NFR-020 | API-004 起 | 无 UI（API） |
| AC-006 | PG-001 存在次要链「紧急管理员登录」，点击进入 break-glass 表单；L2 可见「不用于日常调查」。 | BR-011 | — | PG-001 **Default**；PG-002 **Default** |
| AC-007 | 错误用户名或错误密码提交后均同一失败反馈，响应不包含「用户不存在」与「密码错误」的区分文案。 | BR-011 | API-003 `HNT-AUTH-1003` | PG-002 **Error** 或 **Success-Failure** |
| AC-008 | break-glass 登录成功后落地告警列表，且壳上常驻「非日常调查」类横幅；`is_break_glass=true`。 | BR-011 | API-003、API-004 | PG-003 **Default** |
| AC-009 | 提交 break-glass 后至返回前，提交按钮不可再次发出第二次有效创建会话（连点只产生一个会话）。 | BR-011 | API-003 + Idempotency-Key 可选 | PG-002 **Submitting** |
| AC-010 | 已登录壳展示显示名（可空则角色文案）与角色 `sre` 或 `developer`；不出现第三种人操作角色名称（NOC / 经理）。 | BR-012 | API-004 | NAV-000 叠于 PG-003 **Default** |
| AC-011 | 产品内不存在 Organization 切换器控件。 | BR-002、NFR-016 | 无多组织 API | PG-003 **Default** |
| AC-012 | 产品内不存在「我的团队」自助勾选 / 加入团队控件。 | BR-014 | 无自助 TeamBinding API | PG-003 **Default**；开发直链 PG-011 → **No Permission** |

### 6.2 一期 — 告警列表、深链、详情

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-013 | SRE 账号在无筛选下列出的行包含 `unscoped=true` 的 Alert（当库中存在该类 Alert 时）。 | BR-017、BR-022 | API-010 | PG-003 **Default** |
| AC-014 | 开发账号列表中 **零条** `unscoped=true` 行；用 query 增加 `unscoped`/`team`/`namespace` 扩大可见性后仍不出现未绑定行。 | BR-017、BR-022 | API-010 忽略扩大参数 | PG-003 **Default** 或 **Empty** |
| AC-015 | 开发对不可见 Alert 的深链：页面为 No Permission，正文 **不出现** 该 Alert 的 `alertname`。 | BR-017、BR-022 | API-011 `HNT-AUTHZ-2001` | 当前页 **No Permission**（不得落 PG-004） |
| AC-016 | 合法 `cluster_id`+fingerprint 且库中无行：L1 文案 **精确包含**「尚未收到 AM webhook」，且不渲染伪造的 alertname / 标签卡。 | BR-051 | API-011 `HNT-RES-5001` | PG-004 **Default** |
| AC-017 | 深链缺 `cluster_id` 或 fingerprint：可见「链接不完整」，仍不伪造 Alert。 | BR-051 | `HNT-VAL-3002` | PG-004 **Error** |
| AC-018 | 命中且可见的深链打开告警详情，展示同一 `cluster_id` 与 fingerprint。 | G1、BR-051 | API-011 | PG-005 **Default** |
| AC-019 | 未登录打开深链：先完成登录再回到同一 `cluster_id`+fingerprint 路由（成功则 AC-018，未入库则 AC-016）。 | BR-010、BR-051 | API-001 `return_to` | PG-001 **Default** → 其后 PG-004 或 PG-005 |
| AC-020 | 打开可见 PG-005 时写入 HuntAI「看过」审计；页面 **无**「标记已读」按钮，**无**「已同步到 Alertmanager / xMatters」文案。 | BR-045、BR-044 | API-012 | PG-005 **Default** |
| AC-021 | `llm_available=false` 时 PG-003 仍列出 firing 行，并出现「AI 调查不可用，告警列表仍有效」类横幅；告警行不因此被隐藏。 | BR-026、BR-050 | API-004、API-010 | PG-003 **Default**（横幅叠 Default） |
| AC-022 | 可见范围为 0 且用户未改筛选：空态，不是 Error。 | BR-017 | API-010 空 `items` | PG-003 **Empty** |
| AC-023 | 作用域内有数据但当前筛选 0 条：展示可清除筛选的空结果，不是 Empty 文案混用。 | BR-017 | API-010 | PG-003 **No Result** |
| AC-024 | 列表请求失败：页内 Error，文案不含栈 / SQL / 密钥。 | IX-FB-03 | `HNT-SYS-9001` | PG-003 **Error** |
| AC-025 | PG-003 行操作只有进入详情；列表 **无**「调查」按钮。 | BR-030、BR-041 | — | PG-003 **Default** |
| AC-026 | PG-005 **不渲染** Silence、Ack、Page、IM、邮件、Restart、Scale 按钮或「即将上线」入口。 | BR-044 | 无对应 API | PG-005 **Default** |
| AC-027 | 关闭 LLM 时 PG-005 仍展示 firing 告警字段与「调查」入口（调查行为见 AC-048）。 | BR-026、BR-050 | API-012 | PG-005 **Default** |
| AC-028 | 列表加载过程有 Loading；加载完成后进入 Default / Empty / No Result 之一，不停留在无数据的空白白屏且无状态。 | BR-017 | API-010 | PG-003 **Loading** → 终态 |

### 6.3 一期 — 调查

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-029 | 仅在可见 PG-005 确认后创建 Investigation；webhook 入库成功 **不**因此新建 Investigation（同一 firing 周期查库条数为 0 条 `trigger=human` 以外的自动调查；一期 `trigger` 仅 `human`）。 | BR-030、BR-021 | API-080 不返回 investigation id；API-013 一期无 `auto` | PG-005 **Default**（入库无 UI） |
| AC-030 | 点「调查」必须先出确认层，文案含占用 1 次日额度、墙钟≤150s、只读；确认前可取消且不创建。 | BR-030、BR-041、BR-040 | — | PG-005 **Default**（确认层） |
| AC-031 | 确认成功后以进入 PG-006 为准（有 `investigation_id`）；不以 Toast 代替跳转。 | BR-030 | API-020 201 | PG-005 **Submitting** → PG-006 **Loading** 或 **Default** |
| AC-032 | 创建请求无 `Idempotency-Key` 不得 201。 | BR-030 | API-020 `HNT-VAL-3001` | PG-005 **Error** 或保持确认层 |
| AC-033 | 同一确认连点（同一 Idempotency-Key）只创建 **一条** Investigation。 | BR-030 | API-020 | PG-005 **Submitting** |
| AC-034 | 不同 Idempotency-Key 允许对同一 Alert 再开另一次人工调查，且各占日额。 | 实体 1—N | API-020 | PG-005 **Default** |
| AC-035 | 对不可见 Alert POST 调查：不落 `running` 行、不占日额。 | BR-017、BR-041 | `HNT-AUTHZ-2001` | PG-005 **No Permission** |
| AC-036 | 当日该用户已成功新建 30 个 `trigger=human` 调查后再确认：拒绝文案含「今日调查次数已用尽」；当日计数不因本次拒绝增加。 | BR-041、NFR-012 | `HNT-QUOTA-4001` | PG-005 **Success-Failure** |
| AC-037 | `new_investigations_paused=true` 时确认被拒绝，文案含「组织已关闭新调查」；不占日额。 | BR-042 | `HNT-QUOTA-4002` | PG-005 **Success-Failure** |
| AC-038 | `token_gate_configured=false` 时确认被拒绝，文案含「组织成本闸未配置」；不得暗示可无限调查；不占日额。 | BR-042 | `HNT-QUOTA-4003` | PG-005 **Success-Failure** |
| AC-039 | 人点 `running` 调查数已达 5 时，新的合法确认仍 201 并进入调查页，页面 **不**出现「已达并发上限」拒绝。 | BR-054、NFR-011 | API-020 201 | PG-006 **Default** |
| AC-040 | 创建接口 5xx/503 失败：库中无新 Investigation，当日成功计数不增加。 | BR-041 | `HNT-SYS-9001` / `9002` | PG-005 **Error** 或 **Success-Failure** |
| AC-041 | `llm_available=false` 或禁出域时，确认仍 201，`status=running`，`llm_unavailable=true`；PG-006 横幅文案为「AI 不可用，本次只收集只读证据，不会生成根因结论」。 | BR-053、BR-050 | API-020、API-021 | PG-006 **Default** |
| AC-042 | AC-041 路径终态必须是 `inconclusive`，不得为 `completed`；L1 主标题 **不包含**「根因是」。 | BR-053、BR-032 | API-021 `terminal_reason=no_llm` 或零 citation | PG-006 **Success-Failure**（inconclusive） |
| AC-043 | `completed` 时：每条 Claim 的 `citation_ids` 长度 ≥1；徽章 / 主标题 / 主内容三处与 `inconclusive` 均可被测试员不看接口即区分。 | BR-031、BR-032、G1 | API-021 | PG-006 **Success-Failure**（completed） |
| AC-044 | `citation_count=0` 时 `status` **必须**为 `inconclusive`，不得为 `completed`。 | BR-032 | API-021 | PG-006 **Success-Failure**（inconclusive） |
| AC-045 | Citation 只指向本调查 `outcome=succeeded` 且 `is_readonly=true` 的 ToolCall；响应中不存在客户端写入的 citation id 字段可被 POST。 | BR-031 | API-021、API-022；API-020 body 禁传 Citation | PG-006 **Default**（证据可点） |
| AC-046 | `outcome=failed` 的工具行可见，但点击不能打开 Citation 证据（`citation_id=null`）。 | BR-031 | API-021、API-022 | PG-006 **Default** |
| AC-047 | 调查时间线出现的只读工具名属于闭集：Kubernetes `get`/`describe`/events/pod logs；Prometheus firing/pending 列表；即时 PromQL。一期时间线 **不出现** Loki LogQL、Elastic 查询、Playwright。 | BR-033、BR-035、BR-036 | API-021 | PG-006 **Default** |
| AC-048 | `skill_loaded=false` 时固定展示「未加载组织约束」；调查不因此中断（仍能进入 running/终态）。 | BR-038 | API-021 | PG-006 **Default** |
| AC-049 | 空 Skill Git URL 允许保存且产品可上线；调查表现同 AC-048。 | BR-039 | API-071、API-021 | PG-010 **Empty** 或 **Default**；PG-006 **Default** |
| AC-050 | 单次调查 `llm_call_count` 达到 8 或 `readonly_tool_call_count` 达到 12 或 `wall_clock_ms` 达到 150000 后循环停止；若无 pending 人批则 `status=inconclusive` 且 `terminal_reason` 为 `cost_llm` 或 `cost_tools` 或 `cost_wall_clock`；不得标 `completed`。 | BR-040、NFR-001–003 | API-021 | PG-006 **Success-Failure**（inconclusive） |
| AC-051 | `wall_clock_limit_ms` 恒为 150000，不因配置幻想改变。 | BR-040 | API-021 | PG-006 **Default**（成本闸展示） |
| AC-052 | 无「取消调查」按钮；关页后再次打开同一 id，状态为服务端当前值（running 或终态），不是前端本地清空。 | BR-040 `[ASSUMPTION]` 不可取消 | 无 cancel API；API-021 | PG-006 **Default** 或 **Success-Failure** |
| AC-053 | `truncated=true` 的证据打开后标注「已截断」；API-022 不返回未截断超大原文整包。 | BR-043 | API-022 | PG-006 证据层（从 **Default** 打开） |
| AC-054 | RelatedLink 在调查页标注「外部链接，不是证据」；其 URL **不**出现在任何 Claim 的 `citation_ids` 可解析目标中。 | BR-034 | API-021 `related_links` 不在 `claims` | PG-006 **Default** 或 **Success-Failure** |
| AC-055 | 能列出某 Alert 的用户可打开该 Alert 下 **他人** 发起的 Investigation 只读内容。 | BR-055 | API-013、API-021 | PG-006 **Default** 或 **Success-Failure** |
| AC-056 | 不可见 Alert 的调查 URL：No Permission，不泄露 RCA 正文。 | BR-055 | `HNT-AUTHZ-2001` | PG-006 **No Permission** |
| AC-057 | 调查保留删除后再次 GET：message 语义为「调查已按保留策略删除」，页面不伪造历史 RCA。 | BR-060 | `HNT-RES-5003` | PG-006 **Empty** |
| AC-058 | 用户粘贴文本若被存储，标记为 `user-supplied` 且不得成为 Citation；一期无「Playwright 附件」入口。 | BR-037 | 无粘贴-转-Citation API | PG-006 **Default**（无该入口即 Pass） |
| AC-059 | 375px 视口下 PG-005「调查」主按钮仍可见可点；确认文案仍含「只读」字样（不因窄屏省略）。 | G1、交互 §5 | API-020 | PG-005 **Default** / 确认层 |
| AC-060 | 375px 视口下 `inconclusive` 的 L1 在首屏可见，且仍禁止「根因是…」。 | BR-032 | API-021 | PG-006 **Success-Failure**（inconclusive） |

### 6.4 一期 — 扫描

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-061 | SRE 选择已注册 `cluster_id` 后点「分析该集群」，确认层可见「零 LLM、整集群」；成功创建 `trigger=manual`、`status=running` 的 ScanRun。 | BR-027 | API-030 | PG-007 **Submitting** → 历史行 running |
| AC-062 | 该次扫描成功结束后 PG-008 展示 Finding 快照；页面 **无** Finding ack/close 控件，**无**「据此自动开调查」。 | BR-027 | API-032 | PG-008 **Default** |
| AC-063 | 扫描成功但可见 Finding=0：Empty，不是 `scan_failed`。 | BR-027 | API-032 `findings=[]` | PG-008 **Empty** |
| AC-064 | 集群不可达或扫描器错误：`status=scan_failed`。 | BR-027 | API-032 | PG-008 **Success-Failure** |
| AC-065 | 一期开发主导航 **不出现**「扫描」；直链 PG-007 为整页 No Permission。 | BR-027、§4 | `HNT-AUTHZ-2002` 或 `2001` | PG-007 **No Permission** |
| AC-066 | POST 扫描 body 含 `namespace` 过滤字段不得 201；UI **无** namespace 选择器。 | BR-027 | API-030 | PG-007 **Default** / **Editing**（无该控件即 Pass） |
| AC-067 | 同一集群已有 `running` ScanRun 时再发起：409 `HNT-CONFLICT-5101`；UI 禁用再分析或展示业务失败。 | BR-027、C-PRD-06 | API-030 | PG-007 **Success-Failure** 或按钮禁用叠 **Default** |
| AC-068 | 扫描路径零 LLM：该 ScanRun 生命周期内不得产生 LLM 调用计数（与调查闸无关的独立断言：扫描 API 成功且组织 LLM 关闭时仍 201）。 | BR-027、BR-050、G3 | API-030 | PG-007 **Default** |
| AC-069 | 不存在「启动 24/7 LLM Operator 巡检」的按钮或 API。 | BR-028 | 无该端点 | 全产品（抽查 PG-007 **Default**） |

### 6.5 一期 — 设置、凭证、入库、保留、边界

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-070 | 开发导航 **不出现**「设置」；直链 PG-009–012 为 No Permission。 | §4 | `HNT-AUTHZ-2003` | PG-009 **No Permission**（009–012 同） |
| AC-071 | SRE 可列出并注册 ClusterSource；`cluster_id` 必填唯一。 | BR-004、BR-023 | API-041、API-040 | PG-009 **Editing** → **Submitting** → **Default** |
| AC-072 | 已有 5 个 ClusterSource 时「注册集群」不可再成功新增第 6 个。 | BR-004、NFR-013 | `HNT-VAL-3005` | PG-009 **Default**（按钮禁用）或 **Success-Failure** |
| AC-073 | 0 个集群时为空态，不是 Error。 | BR-004 | API-040 空列表 | PG-009 **Empty** |
| AC-074 | 注册/PATCH 请求出现 kubeconfig 或文件字段 → 失败；页面 **无** kubeconfig 上传控件。 | BR-046、NFR-021 | `HNT-VAL-3006` | PG-009 **Default**（无控件即 Pass） |
| AC-075 | ClusterSource 成功响应只含 `*_configured` 布尔，**不回显**密钥明文。 | BR-052、BR-016 | API-041–044 | PG-009 **Default** |
| AC-076 | 轮换 IngestCredential 成功响应不含 `secret` 字段。 | BR-016、BR-052 | API-044 | PG-009 **Success-Failure** 或 **Default** |
| AC-077 | 一期 PG-009 **不出现** `env` / `write_remediation` / `scale_max` / Loki / WriteIdentity 表单项；PATCH 若带这些字段 → `HNT-CAP-7001`。 | 边界 §4、BR-081 | API-043 | PG-009 **Default** / **Editing** |
| AC-078 | AM webhook 使用 IngestCredential；仅带用户 Session、无 ingest 凭证的 POST 不得 upsert。 | BR-016 | API-080 `HNT-INGEST-6001` | 无 UI |
| AC-079 | 缺 `cluster_id`（或无法对应 ClusterSource）的 webhook 被拒绝，且不留下半行 Alert。 | BR-023 | `HNT-INGEST-6002` / `6005` / `6003` | 无 UI |
| AC-080 | 凭证错误的 webhook 被拒绝，响应不回显密钥、不回告警正文。 | BR-016 | `HNT-INGEST-6001`；成功 202 仅 `{status:upserted}` | 无 UI |
| AC-081 | 同一 `cluster_id`+fingerprint 的后续 firing webhook 更新同一 Alert，不新建第二条 Alert。 | BR-020、BR-021 | API-080；API-010 该身份一行 | 无 UI；SRE 可在 PG-003 **Default** 核对一行 |
| AC-082 | payload 无 `fingerprint` 时用 `alertname`+稳定标签哈希，稳定集不含 `startsAt` 与 `value`（用两次仅 `startsAt`/`value` 不同的 firing 仍落同一 fingerprint）。 | BR-020 | API-080 | 无 UI |
| AC-083 | 缺 `team`/`owner`/`namespace` 三者时仍入库且 `unscoped=true`；开发列表不可见该行；SRE 可见。 | BR-022 | API-080、API-010 | PG-003 **Default**（SRE）；开发 **Empty** 或不含该行 |
| AC-084 | 不存在以个人邮箱 / IMAP / 解析 xMatters 转发信为源的配置页或 API。 | BR-025 | 无该端点 | PG-009 **Default**（无入口即 Pass） |
| AC-085 | 不存在编辑 Alertmanager 规则、删除/改写 xMatters receiver 的 API 或页面。 | BR-001、BR-005、BR-024 | 无该端点 | 抽查设置页 **Default** |
| AC-086 | 不存在 Prom/AM 替换存储、写 recording rule、写 AM silence 的 API。 | BR-001、BR-044 | API 1.4 禁止表 | PG-005 **Default**（无 silence） |
| AC-087 | PG-011：SRE 可把 IdP 组映射到 `team`/`owner`/`namespace` 值；保存须确认；写入后可见最近变更（谁、何时、授予哪些值）的最小化摘要。 | BR-013、BR-014、BR-015 | API-050–052 | PG-011 **Editing** → **Submitting** → **Default** |
| AC-088 | 空映射或空标签值不得保存成功；禁止空映射冒充全员可见。 | BR-013、BR-022 | `HNT-VAL-3007` | PG-011 **Editing**（表单错误） |
| AC-089 | 开发无「加入团队」入口；直链 No Permission。 | BR-014 | `2003` | PG-011 **No Permission** |
| AC-090 | PG-012 Default 在闸未配置时 **必须**显示「闸未配置，不得视为无限」；不得出现「无限」作为可用额度。 | BR-042 | API-070 `token_gate_configured=false` | PG-012 **Default** |
| AC-091 | 「关闭新调查」必须确认；未确认不改变 `new_investigations_paused`。 | BR-042 | API-071（未确认不发请求） | PG-012 确认层 → **Submitting** → **Default** |
| AC-092 | 临时放行必须先指定用户；未选用户不得提交成功。 | BR-041 | `HNT-VAL-3008` | PG-012 **Editing**（表单错误） |
| AC-093 | 未批准放行数字前：API-061 空 body 不得把额度写成 +10/24h；返回 `HNT-QUOTA-4004`。PG-012 **不展示**「+10 次 / 24h」。 | BR-041、C-PRD-04 | API-061 | PG-012 **Success-Failure** 或表单错误叠 **Editing** |
| AC-094 | 开发访问 PG-012 → No Permission。 | BR-041 | `2003` | PG-012 **No Permission** |
| AC-095 | RelatedLink 保存绝对 URL 且 kind ∈ grafana/loki/kibana/elastic 则 201，详情列表可见；空或非 URL 拒绝且不落库。 | BR-034 | API-014 `3003` | PG-005 **Editing** → **Submitting** → **Default** 或内联错误 |
| AC-096 | 无 RelatedLink 删除/编辑 API；UI 未拍板删除则只提供添加与只读列表。 | BR-034 `[TBD-BIZ]` | 无 DELETE related-links | PG-005 **Default** |
| AC-097 | 已部署实例配置的个人 kubeconfig 份数为 0（抽查部署配置与 PG-009 无上传）。生产只读 SA 失败时产品不得改走个人 kubeconfig（无该 fallback 开关）。 | BR-046、BR-047、BR-048、NFR-021 | 无 fallback API | PG-009 **Default** |
| AC-098 | 脱敏字段清单未定时：不得把调查标为 `completed` 的出域 LLM 路径（与 AC-041/042 同时成立：无清单则不得出域完成 RCA）。 | BR-049、NFR-025 | API-021 | PG-006 **Success-Failure**（inconclusive）或 **Default** 横幅 |
| AC-099 | 调查保留库 / Citation 副本 / 应用日志抽查：不出现明文 token / 密钥；Error 页不含 Prompt 原文。 | BR-052、NFR-026 | 任意失败信封 | PG-006 **Error** 或 **Default** |
| AC-100 | Investigation + ToolCall 副本 + Citation 保留 90 天后删除（用测试时钟或过期夹具验证 GET → `5003`）。 | BR-060、NFR-030 | API-021 | PG-006 **Empty** |
| AC-101 | AuditEvent 保留 365 天到期删除（夹具验证；设置页不得靠已删审计伪造历史）。 | BR-061、NFR-031 | 无独立审计列表 API | PG-011 **Default**（最近变更不含已删事件） |
| AC-102 | Alert 实例保留 90 天到期删除；删除后深链走未入库而非伪造卡片。 | BR-062、NFR-032 | API-011 `5001` | PG-004 **Default** |
| AC-103 | 不存在移动端 App 包或「下载 App」入口；Web 为唯一产品入口。 | MVP-20、§9.32 | — | PG-001 **Default** |
| AC-104 | 不存在 VM / SSH / 云主机工具入口。 | BR-003 | 无该 API | PG-006 **Default**（工具名闭集同 AC-047） |
| AC-105 | 读 secrets、kubectl delete/apply/exec、服务端 Playwright：无按钮、无 API。 | BR-044、BR-036、NFR-022、NFR-023、NFR-024 | API 1.4 | PG-005 / PG-006 **Default** |
| AC-106 | 无独立对话首页、无无 `alert_id` 的 Ask、无知识库上传 UI。 | BR-089、§9.24–26 | 无 ChatSession API | 导航抽查 PG-003 **Default** |
| AC-107 | PG-010 允许空仓库；填写则须 Git URL 句法，非法 URL 不得保存。 | BR-038、BR-039 | API-071 | PG-010 **Editing** / **Empty** |
| AC-108 | 列表筛选只允许集群与 `firing`/`resolved`；**无** alertname 全文搜索框（本期不做）。 | 交互 TF-04 `[TBD-BIZ]` | API-010 无 q= | PG-003 **Default** |
| AC-109 | 一期客户端调用 API-023/090/091 得到 404 或 `HNT-CAP-7001`，UI 不出现「即将上线」。 | 边界 §2、API §1.3 | API-023/090/091 | PG-006 **Default**（无人批区、无继续查） |
| AC-110 | 注册账号结构按单 Organization；不提供部门级多租户切换。 | BR-002、NFR-010、NFR-016 | API-070 无 org 列表 | NAV-000 叠 PG-003 **Default** |

### 6.6 〔二期〕验收（独立；一期未交付不得用本表充数）

| ID | Pass 判定 | BR | API | 页面 / 状态 |
|---|---|---|---|---|
| AC-201 | firing 且标签 `severity` **精确等于** `critical` 时，系统可创建至多 1 个本 firing 周期的 `trigger=auto` Investigation。 | BR-070、BR-071 | API-080 无额外 HTTP；API-013 出现 `auto` | PG-005 **Default** |
| AC-202 | 缺 `severity` 键、空值、`Critical`/`CRITICAL`/`warning`：不创建 auto 调查；PG-005 **不**提示「将为你自动开查」。 | BR-070 | API-013 无新 auto | PG-005 **Default** |
| AC-203 | 同 fingerprint 在 resolved 前的重复 firing webhook 不第二次 auto 开查。 | BR-071、BR-021 | API-013 该周期 1 条 auto | PG-005 **Default** |
| AC-204 | 全组织 `trigger=auto` 且 `running` 已达 5：跳过本次自动开查，Alert 仍入库，写 AuditEvent；SRE 在详情 L3 见「自动开查已跳过（并发上限）」；**无**「补开自动」按钮。 | BR-072、NFR-017 | AlertDetail `auto_investigate_skipped_reason=concurrency_cap` | PG-005 **Default**（L3 叠 Default，不是独立 Success-Failure） |
| AC-205 | 组织关闭新调查或闸未配置：跳过 auto 并审计；SRE 可见与并发上限 **可区分** 的「组织闸」原因句。 | BR-042、BR-070 | `org_gate` | PG-005 **Default**；PG-012 **Default** 说明「同时阻断自动开查」 |
| AC-206 | auto 调查 `initiator_principal=auto-investigator`；打开它 **不**增加当前用户当日 human 计数。 | BR-073、BR-041 | API-013、API-004 剩余次数不变 | PG-006 **Default**（徽章 `auto`） |
| AC-207 | `auto-investigator` 不出现登录入口，不出现「以自动调查身份确认」可点主体。 | BR-073、BR-082 | 无该主体 HTTP | PG-001 **Default**；PG-006 人批区 |
| AC-208 | auto 调查零 citation 或触成本闸时必须 `inconclusive`，界面不得像已找到根因。 | BR-074、BR-032 | API-021 | PG-006 **Success-Failure**（inconclusive） |
| AC-209 | 有 LokiCredential 的集群：调查时间线可出现成功 LogQL，且其 `citation_id` 可打开副本。 | BR-075 | API-021、API-022 | PG-006 **Default** |
| AC-210 | 无 LokiCredential：时间线无 Loki 调用，不假装已查日志；无浏览器扒页入口。 | BR-077、BR-036 | API-021 | PG-006 **Default** |
| AC-211 | 开发无 TeamBinding.`namespace`：不得成功跑 Loki；失败行不可 Citation。 | BR-076 | API-021 失败行 | PG-006 **Default** |
| AC-212 | 开发 LogQL 去掉 namespace 约束则拒绝执行，不可 Citation。 | BR-076 | 工具失败 | PG-006 **Default** |
| AC-213 | 不存在 Elastic 查询 API 或 UI 入口。 | BR-075、BR-035 | 无该端点 | PG-006 **Default** |
| AC-214 | Loki **界面 URL** 仍走 RelatedLink 分区；LogQL **查询结果**走 Citation 分区，两区可区分。 | BR-034、BR-075 | API-014 vs API-021 | PG-006 **Default** |
| AC-215 | 开发主导航出现「扫描」但 **无**「分析该集群」；直调 API-030 → 403。 | BR-078、BR-079 | API-030 `2002` | PG-007 **Default** 或 **Empty** |
| AC-216 | 定时 ScanRun `trigger=scheduled` 出现在历史中；间隔按每 ClusterSource 60 分钟（夹具或调度记录）。 | BR-078、NFR-018 | API-031 | PG-007 **Default** |
| AC-217 | 开发只看到 Finding.namespace 与其绑定相交的行；`findings_visible_count` 为过滤后计数。 | BR-079 | API-032 | PG-008 **Default** 或 **Empty** |
| AC-218 | 开发无 namespace 绑定：扫描入口 Empty，说明需管理员绑定 namespace。 | BR-079 | API-031 | PG-007 **Empty** |
| AC-219 | 开发不能把 UI 隐藏理解成「我触发了扫描」：无「运行中且由我触发」的假主体。 | BR-079 | API-031 只读 | PG-007 **Default** |
| AC-220 | `write_remediation` 关或未配 WriteIdentity：PG-006 **整块不出现**人批区（禁止置灰「即将开通」）。 | BR-081 | API-021 `write_actions=[]` | PG-006 **Default** |
| AC-221 | 人批区出现时必须有 preview（集群、`env`、namespace、资源、动词；scale 含当前与目标副本）；缺 preview 不得出现可点「确认执行」。 | BR-082 | API-021 `preview` | PG-006 **Default**（人批 pending） |
| AC-222 | 模型不能自批：不存在不经 API-090 的执行；确认必须是登录有权人的第二次点击。 | BR-082 | API-090 | PG-006 **Submitting**（confirm） |
| AC-223 | SRE/break-glass 可 confirm 任意 `env`；开发在 `production`、未填 env（视为 production）或 namespace 不在绑定时：无确认或 No Permission。 | BR-083、BR-087 | API-090 `HNT-AUTHZ-2004` | PG-006 **No Permission** 或按钮禁用叠 **Default** |
| AC-224 | scale 目标 > 当前×2 或 > `scale_max` 或当前 replicas=0 仍拉起：拒绝执行。 | BR-084、NFR-019 | `HNT-CONFLICT-5104` | PG-006 **Success-Failure** |
| AC-225 | `scale_max` 未配置：该集群禁止 scale；UI 不出现可确认的 scale，或确认禁用原因为未配置；**禁止**填假默认整数。 | BR-084 | `5104` | PG-006 **Default** / **Success-Failure** |
| AC-226 | 有权人点「拒绝」：`rejected`，不执行写。 | BR-082 | API-091 | PG-006 **Success-Failure**（rejected） |
| AC-227 | Investigation 离开 `running` 后 pending → `void`，确认按钮不可点。 | BR-085 | API-090 `5105` | PG-006 **Success-Failure**（void） |
| AC-228 | 连点确认只执行一次。 | BR-082 | API-090 Idempotency-Key | PG-006 **Submitting** |
| AC-229 | WriteIdentity 失败：禁止 fallback 个人 kubeconfig 或只读 SA；可见「写身份不可用」。 | BR-086、NFR-027 | `[ASSUMPTION]` 转 rejected 或 9002 | PG-006 **Error** 或 **Success-Failure** |
| AC-230 | 不渲染 delete/apply/exec 或改 YAML 动词。 | BR-080、BR-044 | `verb` 仅 restart/scale | PG-006 **Default** |
| AC-231 | 写执行结果不可选为 Claim Citation；人批成功不得把 L1 升级成「根因已修复」。 | BR-088 | API-022 对写结果 `5006` | PG-006 **Success-Failure**（confirmed）仍按 RCA 规则 |
| AC-232 | 只读循环已停且存在 pending：`status` 仍为 `running`，`awaiting_write_approval=true`，`wall_clock_ms` 不再增加；L1 文案 **精确为**「只读调查已结束，等待确认写操作」；L1 **不包含**「正在分析」或「正在调查」。 | BR-090、BR-040 | API-021 | PG-006 **Default**（running + 人批等待，不是 Success-Failure） |
| AC-233 | 「继续查」提交后仍停留 PG-006 且绑定原 `alert_id`；不得创建无 Alert 会话。 | BR-089 | API-023；`HNT-VAL-3010` | PG-006 轻量 **Editing** → **Default** |
| AC-234 | 一期之后才启用：PG-009 必选 `env` 为 `production` 或 `non_production`；无「从集群名猜测」按钮；未选 env 不得静默保存为 production 而不告知。 | BR-087 | API-043 `HNT-VAL-3009` | PG-009 **Editing** |
| AC-235 | 开 `write_remediation` 但 `write_identity_configured=false`：保存可成功，页面警告「写身份未配置，调查中不会出现修复」。 | BR-081 | API-043 | PG-009 **Default** |
| AC-236 | 只读 SA 上无「勾选写动词」控件；WriteIdentity 无 kubeconfig 上传。 | BR-047、BR-086 | `3006` | PG-009 **Editing** |
| AC-237 | 开发仍不可见设置导航。 | BR-083、§4 | `2003` | PG-009 **No Permission** |
| AC-238 | 同一 Alert 已有 auto 调查 **不阻止** 再开人工调查。 | BR-030、BR-073 | API-020 201 | PG-005 **Submitting** → PG-006 |
| AC-239 | 工具次数含 Loki，仍 ≤12；auto 调查同样受 8/12/150s。 | BR-074、BR-040 | API-021 | PG-006 **Success-Failure**（若触闸则 inconclusive） |
| AC-240 | 定时扫描 `scan_failed` 时开发不能重跑；SRE 可手动再开。 | BR-078 | 开发无 API-030 | PG-008 **Success-Failure**；PG-007 开发无按钮 |

### 6.7 BR 覆盖矩阵

一期 BR → AC（至少一条）：

| BR | AC |
|---|---|
| BR-001 | AC-085、AC-086 |
| BR-002 | AC-011、AC-110 |
| BR-003 | AC-104 |
| BR-004 | AC-071、AC-072、AC-073 |
| BR-005 | AC-085、AC-020 |
| BR-010 | AC-001–AC-005、AC-019 |
| BR-011 | AC-006–AC-009 |
| BR-012 | AC-010 |
| BR-013 | AC-087、AC-088 |
| BR-014 | AC-012、AC-087、AC-089 |
| BR-015 | AC-087 |
| BR-016 | AC-075、AC-076、AC-078、AC-080 |
| BR-017 | AC-013、AC-014、AC-015、AC-022 |
| BR-020 | AC-081、AC-082 |
| BR-021 | AC-029、AC-081 |
| BR-022 | AC-013–AC-015、AC-083、AC-088 |
| BR-023 | AC-071、AC-079 |
| BR-024 | AC-085 |
| BR-025 | AC-084 |
| BR-026 | AC-021、AC-027 |
| BR-027 | AC-061–AC-068 |
| BR-028 | AC-069 |
| BR-030 | AC-025、AC-029–AC-031 |
| BR-031 | AC-043、AC-045、AC-046 |
| BR-032 | AC-042–AC-044、AC-060 |
| BR-033 | AC-047 |
| BR-034 | AC-054、AC-095、AC-096 |
| BR-035 | AC-047、AC-109 |
| BR-036 | AC-047、AC-105 |
| BR-037 | AC-058 |
| BR-038 | AC-048、AC-107 |
| BR-039 | AC-049、AC-107 |
| BR-040 | AC-030、AC-050、AC-051、AC-052 |
| BR-041 | AC-030、AC-036、AC-040、AC-092、AC-093 |
| BR-042 | AC-037、AC-038、AC-090、AC-091 |
| BR-043 | AC-053 |
| BR-044 | AC-020、AC-026、AC-086、AC-105 |
| BR-045 | AC-020 |
| BR-046 | AC-074、AC-097 |
| BR-047 | AC-097 |
| BR-048 | AC-097 |
| BR-049 | AC-098 |
| BR-050 | AC-021、AC-041、AC-068 |
| BR-051 | AC-016–AC-019 |
| BR-052 | AC-075、AC-076、AC-099 |
| BR-053 | AC-041、AC-042 |
| BR-054 | AC-039 |
| BR-055 | AC-055、AC-056 |
| BR-060 | AC-057、AC-100 |
| BR-061 | AC-101 |
| BR-062 | AC-102 |

二期 BR → AC：

| BR | AC |
|---|---|
| BR-070 | AC-201、AC-202、AC-205 |
| BR-071 | AC-201、AC-203 |
| BR-072 | AC-204 |
| BR-073 | AC-206、AC-207、AC-238 |
| BR-074 | AC-208、AC-239 |
| BR-075 | AC-209、AC-213、AC-214 |
| BR-076 | AC-211、AC-212 |
| BR-077 | AC-210 |
| BR-078 | AC-215、AC-216、AC-240 |
| BR-079 | AC-215、AC-217–AC-219 |
| BR-080 | AC-230 |
| BR-081 | AC-220、AC-235 |
| BR-082 | AC-221、AC-222、AC-226、AC-228 |
| BR-083 | AC-223 |
| BR-084 | AC-224、AC-225 |
| BR-085 | AC-227 |
| BR-086 | AC-229、AC-236 |
| BR-087 | AC-223、AC-234 |
| BR-088 | AC-231 |
| BR-089 | AC-106、AC-233 |
| BR-090 | AC-232 |

NFR 已拍板项由对应 BR 的 AC 覆盖（NFR-001–003→AC-050；NFR-012→AC-036；NFR-013→AC-072；NFR-016→AC-011；NFR-017→AC-204；NFR-018→AC-216；NFR-019→AC-224；NFR-020→AC-001；NFR-021–024→AC-097/105；NFR-025→AC-098；NFR-026→AC-099；NFR-027→AC-229；NFR-028→AC-230；NFR-030–032→AC-100–102）。NFR-004/005/006/014/015/033 及可用性百分比仍 `[TBD-BIZ]`：**不得**用本 PRD 对外宣称 SLA；AC-038/090 覆盖「闸未配置不得视为无限」。

---

## 7. Out of Scope

### 7.1 永不做（Business Model §9 共 40 项，不改写）

1. 替换 Prometheus 或自建指标存储。
2. 替换 Alertmanager 或在 HuntAI 内编辑告警规则。
3. 替换 xMatters 排班、升级、电话、短信。
4. HuntAI 发起 page / 升级 / 语音 / 短信。
5. 对 Alertmanager 写 silence。
6. 对 xMatters 写 ack、关单、评论、附件。
7. 出站 IM：钉钉、飞书、Slack、企业微信。
8. 出站邮件通知。
9. IMAP / 收取个人邮箱 / 解析 xMatters 转发信。
10. kubectl 写：delete Pod、apply YAML、exec。
11. 读 Kubernetes secrets。
12. 对任意 firing 入库即自动开查（无 `severity=critical` 门槛）。
13. 把缺 `severity` 键当成自动开查放行。
14. 24/7 LLM Operator 巡检。
15. Elasticsearch 官方查询 API 作为调查工具。
16. 服务端 Playwright / 浏览器登录 Elastic 或 Grafana Loki 扒日志。
17. 把用户粘贴日志当 Citation。
18. 已部署 HuntAI 使用个人 kubeconfig（含共享「开发套」上传 kubeconfig）。
19. 本机 HuntAI 使用个人 kubeconfig 对准生产集群。
20. ServiceAccount 或 WriteIdentity 失败时 fallback 个人 kubeconfig。
21. VM、SSH、云主机、非 Kubernetes 中间件工具。
22. 自建服务目录 / CMDB。
23. 与 xMatters 组绑定作为权限主键。
24. SuperBizAgent 式 RAG 对话（无 `alert_id` 的 `diagnose()` / 快速聊天首页）。
25. 知识库文件上传 UI。
26. 无 Alert、仅选集群的 Ask。
27. Keep 企业版闭源 AI Correlation 的复刻。
28. 完整 IRM：事故对象、on-call 日历、状态页。
29. OneUptime 式探针、APM、日志/Trace 存储。
30. GitHub / GitLab 自动开 PR、auto-merge。
31. Jira / 工单系统开单。
32. 移动端 App。
33. 对外 SaaS 多 Organization。
34. 部门级多租户。
35. NOC 角色、经理操作角色。
36. 开发 on-call 触发整集群扫描或 namespace 扫描。
37. 用户自助勾选 TeamBinding。
38. 以未评测的 MTTR / 准确率数字作为产品承诺。
39. Slack / 其它 IM 作为第一入口。
40. Prometheus 写 recording rule / 写告警规则。

交互 0.3：上述能力任何阶段都不得以按钮、菜单、表单或「即将上线」入口出现。Webhook 入库无「入库监控台」。

### 7.2 一期相对二期：本期不做（二期独立验收）

- 自动开查徽章 / 「打开自动调查」作为一期验收项
- 调查内 LogQL 工具条与 Loki Citation
- ApprovalRequest 人批卡片、Restart/Scale
- 定时 ScanRun、开发只读扫描导航
- ClusterSource `env` / `write_remediation` / `scale_max` / LokiCredential / WriteIdentity
- API-023 / API-090 / API-091
- 审批中心页、对话首页（二期也永不做独立对话产品）

### 7.3 API 明确不提供（直至产品改 BR 并走 change_log）

取消调查；RelatedLink 删除/编辑；alertname 全文搜索；登出 HTTP（交互无 TF，API §11 A12）；通用 Job；把模型输出的 Citation id 写入。

### 7.4 运营而非本产品功能

- 在 xMatters 模板中加入深链
- 各集群 Alertmanager 打上 `severity` 键（二期自动开查的上线门槛）
- 保留 xMatters 原 receiver 并增加 HuntAI receiver（双发）

---

## 8. 供 Stage 13 `highfi_page_map.md` 的 UI-AC 索引

下列 AC 均含 PG-xxx + 状态名，必须被高保真页映射覆盖。纯 API / webhook 无 UI 的 AC-005、AC-078–AC-082 不要求 highfi 页。

| 页面 | 状态 | AC |
|---|---|---|
| PG-001 | Default | AC-001、AC-002、AC-006、AC-019、AC-103、AC-207 |
| PG-001 | Submitting | AC-002 |
| PG-001 | Error | AC-003、AC-004 |
| PG-002 | Default | AC-006 |
| PG-002 | Submitting | AC-009 |
| PG-002 | Error / Success-Failure | AC-007 |
| PG-003 | Default | AC-008、AC-010–AC-014、AC-021、AC-025、AC-028、AC-083、AC-106、AC-108、AC-110 |
| PG-003 | Empty | AC-014、AC-022、AC-083 |
| PG-003 | No Result | AC-023 |
| PG-003 | Error | AC-024 |
| PG-003 | Loading | AC-028 |
| PG-004 | Default | AC-016、AC-102 |
| PG-004 | Error | AC-017 |
| 当前页 | No Permission | AC-015 |
| PG-005 | Default | AC-018、AC-020、AC-026、AC-027、AC-029、AC-030、AC-059、AC-201–AC-205、AC-238 |
| PG-005 | Submitting | AC-031、AC-033、AC-238 |
| PG-005 | Success-Failure | AC-036–AC-038、AC-040 |
| PG-005 | No Permission | AC-035 |
| PG-005 | Error | AC-032、AC-040 |
| PG-005 | Editing / Submitting | AC-095 |
| PG-006 | Default | AC-031、AC-039、AC-041、AC-045–AC-049、AC-051–AC-056、AC-058、AC-104–AC-105、AC-109、AC-206–AC-207、AC-209–AC-214、AC-220–AC-223、AC-230、AC-232–AC-233 |
| PG-006 | Loading | AC-031 |
| PG-006 | Success-Failure | AC-042–AC-044、AC-050、AC-060、AC-208、AC-224–AC-227、AC-229、AC-231、AC-239 |
| PG-006 | Empty | AC-057、AC-100 |
| PG-006 | No Permission | AC-056、AC-223 |
| PG-006 | Submitting | AC-222、AC-228 |
| PG-006 | Editing | AC-233 |
| PG-006 | Error | AC-099、AC-229 |
| PG-007 | Default / Editing / Submitting | AC-061、AC-066–AC-069、AC-215–AC-216、AC-219 |
| PG-007 | Empty | AC-218 |
| PG-007 | No Permission | AC-065 |
| PG-007 | Success-Failure | AC-067 |
| PG-008 | Default | AC-062、AC-217 |
| PG-008 | Empty | AC-063、AC-217 |
| PG-008 | Success-Failure | AC-064、AC-240 |
| PG-009 | Default / Editing / Submitting / Empty / Success-Failure / No Permission | AC-070–AC-077、AC-084–AC-086、AC-097、AC-234–AC-237 |
| PG-010 | Default / Empty / Editing | AC-049、AC-107 |
| PG-011 | Default / Editing / Submitting / No Permission | AC-087–AC-089、AC-101 |
| PG-012 | Default / Editing / Submitting / Success-Failure / No Permission | AC-090–AC-094、AC-205 |

---

## 9. 完成前自检

- [x] 每条 AC 可执行 Pass/Fail 判定（无「体验良好 / 响应迅速」）
- [x] 每条 BR（一期 BR-001–BR-062 已列编号 + 二期 BR-070–BR-090）至少被一条 AC 覆盖（§6.7）
- [x] 涉及 UI 可见结果的 AC 已标注 PG-xxx 与状态，并在 §8 汇总供 Stage 13 逐条覆盖
- [x] 未重新定义业务；未把 `[TBD-*]` 填成假默认；未新增 PG-013+ 或 Gate 1 未记载的页面
- [x] 一期与二期验收分离；Out of Scope 含永不做 40 项
- [x] 上游缺口与契约/交互差异已标 `[CONFLICT]`（§0），未自行裁决

**实现禁令**: 不得在未改 Business Model 并走 `docs/13_changes/change_log.md` 的情况下，把 §7 任一项做成功能，或把二期 AC 标进一期 Gate，或把本文 Status 升为「已冻结」而不经用户显式批准。
