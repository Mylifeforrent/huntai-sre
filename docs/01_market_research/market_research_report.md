# HuntAI SRE 业务调研报告

- **Status**: Draft（业务调研阶段产出；非 PRD、非已批准产品定义）
- **日期**: 2026-09-13
- **方法**: 汇总已完成的竞品研究包，不另做桌面市场规模估算。五家材料由并行 Agent 分产品抽取后由本报告收口。
- **研究包（仓库外，禁止当成本仓库阶段产出）**: `../opensource-product-analysis/competitors/sre-agent-analysis/`（以下简称「研究包」；分析日 2026-09-13）
- **标记**: 【事实】研究包已取证；【推断】研究包标「证据推断」；【结论】本报告立项判断；数字无来源则 **[TBD-RESEARCH]**；HuntAI 产品范围 **[TBD-BIZ]**。

**一句话结论（【结论】，待 Stage 8 PRD 确认）**: 值得做的不是又一个「全能 AIOps」，而是给已有监控栈加一层**可信 AI 运维**：规则 finding 可关模型 → 可解释降噪 → 只读 cited RCA → 写操作策略写死并人批。本仓库产品尚未建设；`decision/huntai/huntai-sre` 仍为空（研究包 `07-证据不足清单.md` #11）。

---

## 1. 目标用户与核心痛点

**用户画像（【事实】，研究包拆解口径）**: 开发工程师 / SRE / on-call，用自动化降低告警噪音、缩短根因调查、在可控范围内触发修复。来源：研究包 `modules/00-共享证据基线.md` 文首。

| 切面 | 谁 | 依据 |
|---|---|---|
| 集群排障 | Kubernetes SRE / 平台工程 | K8sGPT `01-产品定位.md`：分析器编码 SRE 经验，CLI 无 Web 控制台 |
| 事故调查 | 云原生 on-call；也可非 K8s（VM/云/库） | HolmesGPT `01-产品定位.md`：`holmes ask` / 告警调查；CNCF Sandbox 2025-10-08 |
| 告警收口 | SRE / NOC；文档对照「~100 人公司买不起传统 AIOps」 | Keep `01-产品定位.md` [S10]（对照叙事，非用户量） |
| 全栈值班 | SRE + 要状态页的产品团队 | OneUptime `01-产品定位.md`：自托管 vs 受监管 Cloud |

HuntAI 的具体客户栈（K8s-only vs 混合云、对接的监控/IM/Git）**[TBD-BIZ]**，须在 Stage 3 后续补充或 Stage 8 确认（研究包 `06-决策建议.md` 仍需验证 #1）。

**痛点（【事实】= 竞品已产品化的问题；幅度均无可靠量化）**:

1. **告警分散与疲劳**: Keep 价值主张为单一面板 + 降噪；「significantly lowering MTTx」无数字（Keep `01` [S6]；赛道 `07` #2）。
2. **根因调查贵、慢、不可复盘**: Holmes 主路径是 agentic RCA；STCLab 单案例自报人工 15–20 分钟 → 读摘要 2 分钟（Holmes `01` [S13]，**不可外推**）。四家均无公开准确率 / 误报率 / MTTx（赛道 `07` #2）。
3. **无 Key / 断网 / 禁止出域时仍要发现故障**: K8sGPT `analyze` 默认可不走模型（`03` [S11]）；OneUptime Insights 15 分钟零 token（`03` [S9]）。
4. **模型乱写集群**: 四家写路径都弱或拆权；Keep 工作流文档缺运行时审批（赛道 `04-能力对比.md` HITL 行）。
5. **LLM 账单与 on-call 一起爆**: OneUptime 将成本闸产品化（per-run 8 LLM / 12 tools / 150s；Cloud **$20 / 1M tokens**，2026-09-13，OneUptime `03` [S7/S9]）。

内部前身 SuperBizAgent（**不是竞品**）把痛点暴露为工程缺口：无登录、诊断任务写死、无 cited、无 HITL、MCP 为 mock（快照 `release-2026-05-17`，`super-biz-agent/README.md`）。

---

## 2. 主要使用场景

开发工程师任务链（【推断】，置信度中，研究包 `modules/00` §三）:

```text
现网告警/集群异常
  → 去重与可解释关联（Keep 指纹/规则/拓扑）
  → 规则 Finding 可关 AI（K8sGPT analyze / OneUptime Insights）
  → 只读 cited RCA（Holmes tool loop / OneUptime 调查）
  → Skills 约束「不要查什么」（Holmes SKILL.md）
  → 写操作策略写死 + 人批（Holmes remediation；OneUptime 调查不能 ack/page）
```

与 HuntAI 相关的四条场景（【结论】= 应覆盖的任务，不是已实现）：

| 场景 | 用户要完成的事 | 竞品锚点 | 默认 vs 非默认 |
|---|---|---|---|
| A 无模型仍能发现 | CrashLoop / ImagePull 类 finding | K8sGPT `analyze`；Insights 15min | AI 可选 |
| B 告警驱动只读调查 | webhook/告警 ID → 白名单工具 → 可点击引用 | Holmes `investigate`；OneUptime 只读 RCA | 调查默认只读 |
| C 降噪后进人审 | fingerprint → 规则（candidate）→ 拓扑 | Keep OSS 三件套 | 全自动 AI 关联 **仅** Keep Cloud/Enterprise |
| D 可控写修复 | restart/scale/开 PR 前必须人批 | Holmes 变异永为人批；OneUptime Fix **无 auto-merge** | 写路径 opt-in；K8sGPT 自动修 Alpha 默认关 |

**营销 ≠ 默认能力（【事实】，研究包交叉核对，2026-09-13）**: Holmes「By design, read-only」≠ 系统不能写；K8sGPT 官网「Automatically apply suggested fixes」≠ 默认；OneUptime「Wake up to pull requests」≠ 调查会改代码；Keep「AIOps 2.0」≠ 开源可用全自动关联。

---

## 3. 市场结构、规模与趋势（简要）

四家**不在同一产品层**（【事实】，赛道 `README.md` / `04-能力对比.md`）：

| 产品 | 层 | 许可 | GitHub API（2026-09-13） |
|---|---|---|---|
| K8sGPT | K8s **规则分析器** + 可选 `--explain` | Apache-2.0 · CNCF Sandbox（入驻 2023-12-19） | 8,175 Star / 1,053 Fork |
| HolmesGPT | **Agentic 调查 Agent**；默认只读 RCA | Apache-2.0 · CNCF Sandbox（接纳 2025-10-08） | 3,283 Star / 481 Fork |
| Keep | **告警 IRM** + YAML 工作流 | MIT（`ee/` 除外） | 12,314 Star / 1,507 Fork |
| OneUptime | **完整可观测 + 事故平台**；AI 为可选层 | Apache-2.0（自称非 open-core） | 7,601 Star / 451 Fork |

**市场规模 / 付费用户 / ARR**: 四家均无可靠公开数字；Holmes ADOPTERS 2 行、K8sGPT ADOPTERS 3 组织；OneUptime/Keep 仅有营销措辞。Star ≠ 用户数（赛道 `07` #1、#12）。**[TBD-RESEARCH]**，禁止用上表 Star 外推 TAM。

**可核验的趋势信号（【事实】）**:

1. **分层商品化**: 规则扫描、告警 IRM、调查 Agent、全栈可观测已分开卖；混装成「一个大脑」与四家文档权限模型冲突。
2. **AI-Free 成为卖点**: K8sGPT 官网 AI-Free Analysis；OneUptime Insights 零 token；Keep 规则/指纹不依赖 LLM。
3. **写修复商业化但默认关**: 自动修 / 自动开 PR / 24/7 LLM 巡检在文档里是 Alpha、opt-in 或企业墙。
4. **Token 被标价**: OneUptime Cloud 全局 LLM **$20 / 1M tokens**（超 10M/月谈销售，折扣 **[TBD-RESEARCH]**）；Keep Growth 公开 **$199/月** 起（托管配额，≠ 企业 AI 成交价）。
5. **CNCF 接纳调查类 Agent**: Holmes Sandbox 2025-10-08，说明「告警→RCA Agent」已被基金会赛道承认，不等于生产就绪（Operator 仍 alpha）。

OneUptime 首页「<30s 检测 / 50% MTTR / 40% 更少工单」无方法口径（`01` [S7]），**不采用**为市场基准。

---

## 4. 机会与风险

**机会（【推断】← 研究包 `06-决策建议.md` 五条借鉴，置信度中）**:

1. 只读调查 + 写路径 opt-in + 策略写死审批（Holmes + OneUptime）。
2. 规则 finding 与 LLM 解耦——无 Key 仍能出结果（K8sGPT + Insights）。
3. 可解释关联三件套，**不抄** Keep `ee/` 专有模型。
4. Skills/`SKILL.md` 把「不要查什么」版本化（Holmes；STCLab 同模型有无 runbook 4.6 vs 3.6，单案例）。
5. 成本闸 + 服务端 cited RCA（OneUptime per-run cap；Holmes 大结果落盘/ulimit）。

默认假设（研究包 `06` MVP 段，**[ASSUMPTION]**）: 告警/指标/日志已有来源；HuntAI 做可信 AI 层，**不新建**探针、状态页或完整 IRM。

**风险**:

| 风险 | 类型 | 依据 |
|---|---|---|
| 做成 OneUptime 第二 | 范围 | 主体工程量在可观测本体；单人不可复制（`06` 暂不建设） |
| 把企业 AI 关联当开源能力卖 | 合规/话术 | Keep STEP 4 ENTERPRISE ONLY；OSS ⛔️ |
| 默认自动 apply | 安全 | K8sGPT Alpha 无审批；Keep 工作流缺运行时 HITL |
| 复制 90+ 集成 / 38+ toolsets | 安全债 | Holmes 0.40.0 修复 injection/SSRF；每个工具都是攻击面 |
| 无评测却承诺 MTTR | 质量 | 赛道 `07` #2；须自建故障注入集 |
| 出域模型合规未定 | **[TBD-BIZ]** | `06` 仍需验证 #7 |
| 继承 Demo 裸奔 | 安全 | SuperBizAgent 无鉴权、CORS `*`、诊断写死 |

**不纳入立项范围（【结论】，对齐 `06` 暂不建设）**: 24/7 LLM Operator 巡检、自动开 PR、MCP 全量 CRUD、完整可观测平台、Keep 企业关联模型。

---

## 5. 为什么是现在？为什么是我来做？

**为什么是现在（【推断】，基于 2026-09-13 取证，不是「市场很大」）**:

1. 四家已经把层拆开，客户不再需要从零解释「Agent ≠ 监控平台」；空白在**层与层之间的治理**（citation、人批、成本闸、AI-Free），而不是再做一个仪表盘。
2. 营销与文档持续打架（四家均有），说明市场话术超前于默认安全模型——后发产品可以用「默认只读 / 默认可关 AI / 默认关写」做差异，而不是比谁更「自动」。
3. Token 已被标价、巡检被标 alpha：现在做成本闸与告警驱动（而非 24/7 烧模型）与领先产品的**文档真相**一致，而不是与首页文案一致。
4. 内部已有 SuperBizAgent 快照（2026-05-17）：规划环与 Prometheus 真查可继承，鉴权/cited/HITL 必须重做——窗口是「把 Demo 做成可信层」，不是「再写一个聊天机器人」。

**为什么是我（具体约束，非市场规模）**:

1. **已经付过研究成本**: 研究包含四家 01–07 + modules，以及 SuperBizAgent **源码级** S1–S29；不是从官网重新猜。
2. **栈连续**: 前身为 FastAPI + LangGraph；本仓库 Stage 2 已选同一熟悉栈（根 `README.md`，未冻结）。单人没有缓冲去重造 TypeScript 全栈可观测（OneUptime 主体）。
3. **范围可砍**: 研究包已写明「不要重建 OneUptime、不要复制集成表」；与单人 SDD 时间盒一致。
4. **有反面教材**: SuperBizAgent 证明「企业级」文案在无登录、无 alert_id、无 cited 时不成立——作者本人踩过，不会把缺口写成卖点。

若 Stage 8 确认必须自建全栈 IRM 或默认自动修复，则本「为什么是我」不成立，应停做或改立项。

---

## 6. 待确认清单（禁止用假默认值填 PRD）

| 标记 | 问题 | 谁 / 何时 |
|---|---|---|
| **[TBD-BIZ]** | 目标客户栈、监控/IM/Git 清单、IRM vs 纯调查切面 | 维护人 · 问题建模 / PRD |
| **[TBD-BIZ]** | 调查是否允许出域（DashScope / 开源模型网关 / 禁出域） | 维护人 · PRD + 架构 |
| **[TBD-RESEARCH]** | TAM/付费用户/ARR；RCA 准确率与 MTTx | 不编造；自建评测集 |
| **[TBD-RESEARCH]** | OneUptime 10M token 以上单价；Keep 企业关联成交价 | 不采信营销页 |
| **[ASSUMPTION]** | 现网已有告警/指标/日志，本产品不新建探针 | Stage 8 必须显式接受或推翻 |

完整竞品对照与 MVP 条款仍以研究包 `04` / `06` 为准；本仓库 Stage 4 约定输出 `docs/02_competitor_analysis/competitor_analysis_report.md` 尚未产出，本报告**不替代**该文件。

---

## 来源索引

| 研究包路径 | 本报告用途 |
|---|---|
| `.../sre-agent-analysis/README.md` | 四层划分、营销交叉 |
| `.../04-能力对比.md` · `06-决策建议.md` · `07-证据不足清单.md` | 矩阵、借鉴/暂不建设、共同空白 |
| `holmesgpt/` · `k8sgpt/` · `oneuptime/` · `keep/` 各 01/05/06/07 | 用户、AI 边界、数字 |
| `super-biz-agent/`（内部基线，非竞品） | 「为什么是我」与禁止带进 MVP |
