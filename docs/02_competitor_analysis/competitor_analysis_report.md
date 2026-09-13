# HuntAI SRE 竞品分析报告

- **Status**: Draft（竞品分析阶段产出；非 PRD）
- **日期**: 2026-09-13
- **输入**: [`docs/01_market_research/market_research_report.md`](../01_market_research/market_research_report.md)
- **范围**: 调研指定的四家开源产品。SuperBizAgent 是内部基线，**不是竞品**。
- **研究包**: `../opensource-product-analysis/competitors/sre-agent-analysis/`（分析日 2026-09-13）
- **标记**: 【事实】研究包或下列 URL；【推测】须标出。无应用商店；「抱怨」只引用带链接的 GitHub Issue / 安全通告，**未做评论区定量统计，故不称最高频**。

**一句话（【结论】）**: 四家不在同一层，HuntAI 的生存空间不在「再做一个」其中任一层，而在把 **AI-Free finding + 告警驱动的只读 cited RCA + 策略写死的人批写路径 + 成本闸** 做成默认可交付的治理层。

---

## 1. HolmesGPT（调查 Agent）

| 项 | 内容 |
|---|---|
| 定位与功能【事实】 | CNCF Sandbox、Apache-2.0。`holmes ask` / `investigate` + Skills/MCP。默认只读 RCA；写修复 Helm addon；Operator **alpha**。研究包 `holmesgpt/README.md` |
| 核心流程【事实】 | 告警或自然语言 → 工具循环 → 逐步 tool 痕迹 → 根因与**建议**。变异 kubectl **永远人批**。`holmesgpt/02-功能地图.md` |
| AI【事实】 | 调查主路径依赖 LLM；无零 token 扫描环。Operator 每 tick ≥1 次 LLM。`holmesgpt/05-AI专项.md` |
| 商业【事实】 | 开源自建；UI/Slack 标 Robusta 3rd party。**无官方价目**。`holmesgpt/01-产品定位.md` |
| 社区反馈（GitHub，非商店） | [Issue #2228](https://github.com/HolmesGPT/holmesgpt/issues/2228)（2026-06-24，@mdecalf）：Helm `config.tools` 未真正做成 allowlist，`merge_pull_request` 仍可被模型调用。 [Issue #2329](https://github.com/HolmesGPT/holmesgpt/issues/2329)：ScheduledHealthCheck 部分 run 早退，停在 “Now investigating”。**厂商自承**（非 Issue）：Slack 按钮调查丢 alert-labels（`holmesgpt/02` S40）。 |
| 借鉴【事实←06】 | 只读默认 + 危险工具策略写死；`SKILL.md` 版本化「不要查什么」；MCP `health_check` 探活。 |
| 没解决 | 官方定价与审计保留；Slack 不能当第一入口；Operator 非生产；无强制 `alert_id` 契约（对比内部 Demo 同样缺入参）。 |

---

## 2. K8sGPT（规则分析器）

| 项 | 内容 |
|---|---|
| 定位与功能【事实】 | CNCF Sandbox、Apache-2.0。CLI 规则扫描；无 Web 控制台。官网 “automatically apply fixes” **不是默认**。`k8sgpt/README.md` |
| 核心流程【事实】 | `analyze` **不调模型** → 可选 `--explain`。Operator 写 `Result` CR。自动修复仅 Operator、默认关、Alpha、无审批、无回滚。`k8sgpt/02-功能地图.md` |
| AI【事实】 | 默认可 AI-Free；解释才 BYOK 出域。CLI `--anonymize` 为显式开关；Events **不脱敏**。`k8sgpt/05-AI专项.md` |
| 商业【事实】 | 无官方 SaaS 价；ADOPTERS 3 家。第三方 “~$10/mo” **不采信**。`k8sgpt/01-产品定位.md` |
| 社区反馈 | README 指向 [Issue #560](https://github.com/k8sgpt-ai/k8sgpt/issues/560)：Event 消息脱敏仍是未来项（研究包 `k8sgpt/03` 结论 19）。[Operator #433](https://github.com/k8sgpt-ai/k8sgpt-operator/issues/433)：LocalAI/Bedrock 下 `Result.details` 为空，多名用户复现。PR [#1713](https://github.com/k8sgpt-ai/k8sgpt/pull/1713) 转述用户：IngressClass 「不存在」实为 RBAC Forbidden **误报**。 |
| 借鉴【事实←06】 | 分析器与 LLM 解耦；修复默认关 + dry-run/RV 门；公开脱敏未覆盖清单。 |
| 没解决 | 告警驱动 RCA、finding 关闭/误报看板、Web 与租户、MCP HTTP 鉴权文档。 |

---

## 3. OneUptime（可观测平台 + 可选 AI 层）

| 项 | 内容 |
|---|---|
| 定位与功能【事实】 | Apache-2.0 全栈监控/事故/状态页。AI SRE **默认 Off**。首页 “Wake up to pull requests” ≠ 调查会改代码。`oneuptime/README.md` |
| 核心流程【事实】 | 探针→事故（无 AI 可闭环）。调查：只读工具 → **服务端 citation** → 时间线；不能 ack/page。Insights 15min **零 token**。Fix Task 开 PR、**无 auto-merge**。`oneuptime/02-功能地图.md` |
| AI【事实】 | 调查只读；Ask AI 默认 Ask for approval；Quiet mode inconclusive 不 page。`oneuptime/05-AI专项.md` |
| 商业【事实】 | Cloud Growth **$22** / Scale **$99**（2026-09-13 定价页）。AI **$20/1M tokens** 或 BYOK。自称非 open-core。`oneuptime/01-产品定位.md` |
| 社区反馈 | 研究包无商店评论。公开安全通告：[GHSA-hm7m-9qjj-xj5x](https://github.com/OneUptime/oneuptime/security/advisories/GHSA-hm7m-9qjj-xj5x) 认证用户可列出**其他项目** LLM provider 清单；[GHSA-r5v6-2599-9g3m](https://github.com/OneUptime/oneuptime/security/advisories/GHSA-r5v6-2599-9g3m) 客户端头可绕过租户隔离（通告针对 v10.0.20；是否仍影响当前 13.x **[TBD-RESEARCH]**）。 |
| 借鉴【事实←06】 | cited RCA；per-run/日 token 闸；Insights 不用 LLM 发现。 |
| 没解决 | 主体工程量在探针与存储——**HuntAI 不应复制**。公开无 RCA 采纳率。 |

---

## 4. Keep（告警 IRM 层）

| 项 | 内容 |
|---|---|
| 定位与功能【事实】 | MIT（`ee/` 除外）。单一面板、去重、规则/拓扑、YAML 工作流。**不是**调查 Agent。`keep/README.md` |
| 核心流程【事实】 | Push 告警 → fingerprint → 规则（可 candidate 人审）/ 拓扑 Processor → Incident → 工作流 Actions。`keep/02-功能地图.md` |
| AI【事实】 | 全自动 AI Correlation **OSS ⛔️ / Cloud+Ent ✅**。OSS：规则+拓扑；工作流 BYO LLM ✅；Semi-auto/Assistant **experimental**。`keep/05-AI专项.md` |
| 商业【事实】 | Startup **$0**；Growth **$199/月** 起（2026-09-13）。企业 AI 谈价。禁止抄 `ee/`。`keep/01-产品定位.md` |
| 社区反馈 | 研究包未摘评论。[Issue #6089](https://github.com/keephq/keep/issues/6089)：`AUTH_TYPE=KEYCLOAK` 下 API Key 无 `token` 导致 webhook 鉴权失败；维护者称曾为特定客户修过但未合入上游。 |
| 借鉴【事实←06】 | 先做关联+工作流切面；指纹→规则→拓扑（不依赖闭源模型）；企业 AI **机制**（租户不混训、阈值）不可抄权重。 |
| 没解决 | 文档层**无**副作用执行期 preview→confirm；OSS 无全自动关联；不做 cited RCA。 |

---

## 5. 横向对比矩阵

| | HolmesGPT | K8sGPT | OneUptime | Keep |
|---|---|---|---|---|
| 产品层 | 调查 Agent | 规则扫描器 | 可观测+事故平台 | 告警 IRM |
| 默认可关 AI | 否 | **是**（`analyze`） | Insights 零 token；调查 Off | 规则/指纹不依赖 LLM |
| 调查 cited | 工具痕迹；非服务端铸造 citation【事实】 | 无 RCA 产品 | **服务端 citation**【事实】 | 非 RCA |
| 写路径 | opt-in + 人批；#2228 显示 allowlist 可漏 | Alpha 自动修、无审批 | Fix 分车道、无 merge | 工作流可 restart；**缺运行时审批文档** |
| 告警入参 | 多源 investigate | 无（集群扫描） | 平台内 Incident/Alert | fingerprint / webhook |
| 租户/鉴权 | HTTP API Key 默认可关（研究包） | 无产品账号 | 有多租户；出现过跨租户通告 | OSS 有 DB/OAuth；Keycloak 在 `ee/` |
| 商业 | 无官方价 | 无 SaaS 价 | 套餐 + token 价 | 托管配额墙 + EE AI |
| 对 HuntAI | 学审批与 Skills | 学 AI-Free | 学闸与 citation，不抄平台 | 学可解释关联，不抄 ee、不抄无闸工作流 |

内部基线 SuperBizAgent：FastAPI+LangGraph Demo；无登录、任务写死、无 cited、MCP mock（`super-biz-agent/README.md`）。只继承规划环与 Prom 只读真查。

---

## 6. 差异化切入点（供 Stage 5 直接建模）

**【结论】** HuntAI SRE = 接**现网**告警/指标/日志的 **可信 AI 运维层**，不是第五个监控平台，也不是「更自动的 Holmes/Keep」。

后续 `business_model.md` 必须能落成规则（编号待 Stage 5 分配）：

1. **Finding 可独立于模型**：无 Key / 禁止出域时仍输出规则 finding（学 K8sGPT/Insights；禁止把 LLM 当发现主环）。
2. **调查入口是告警身份**：请求必须带 `alert_id` 或完整 payload + fingerprint；禁止写死口号式 `diagnose()`（相对 Holmes 渠道散、相对 SuperBizAgent 无入参）。
3. **RCA 服务端 cited**：每条结论只允许绑定**已执行**的只读 tool call；零 citation ⇒ inconclusive，**不 page**（学 OneUptime；补 Holmes 仅有步骤痕迹的缺口）。
4. **写操作策略写死 + 执行期 HITL**：默认无写工具；开放后变异动作 preview→confirm，**模型不可自批**（补 Keep 文档缺口与 K8sGPT 无审批；落实 Holmes 人批且避免 #2228 式 allowlist 失效）。
5. **成本闸默认开**：per-run LLM/tool/时限；日 token；大结果截断（学 OneUptime/Holmes 护栏）。
6. **关联只用可解释层**：fingerprint → 规则（candidate 人审）→ 可选拓扑；**禁止**复刻 Keep `ee/` 全自动模型。
7. **默认鉴权、默认脱敏**：匿名调查禁止上线；出域前可逆变脱敏并公开未覆盖字段（相对 SuperBizAgent CORS `*`、K8sGPT Events #560）。

**明确不做（与调研一致）**: 全栈探针/状态页、24/7 LLM Operator、默认 auto-merge/auto-apply、90+ 集成竞赛。

**【推测】** 若目标客户没有现成监控栈，上述切入点失效，应停做或改立项（**[TBD-BIZ]**）。
