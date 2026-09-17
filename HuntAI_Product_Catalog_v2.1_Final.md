**HUNTAI**

**产品目录**

Product Catalog · v2.0

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 33%" />
<col style="width: 33%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Experience</strong></p>
<p><strong>Workmate</strong></p></th>
<th><p><strong>Domain Systems</strong></p>
<p><strong>Test · SRE</strong></p></th>
<th><p><strong>Shared Capabilities</strong></p>
<p><strong>Knowledge · Data</strong></p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

基于《HuntAI 开发套件生态设计讨论整理》重构

目标：形成可用于产品规划、内部对齐、客户沟通与后续 PRD 拆解的统一产品目录。

**DISCUSSION DRAFT · 2026**

**01 / CATALOG LOGIC**

# **这份目录解决什么问题**

本目录不再按“有多少个 Agent”来罗列产品，而是按四个问题划分边界：谁负责用户体验、谁拥有领域事实、谁提供共享能力、谁连接企业外部系统。这样可以避免 Agent、Skill、MCP、Connector 和 UI 被混成同一层。

|     | **本版重构原则** 产品目录面向“可被用户理解和采购/采用的能力集合”，不等于代码仓库列表，也不等于服务注册表。Platform Core、Memory、Scheduler、MCP 等技术能力需要明确存在，但不必全部成为一级产品。 |
|-----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## **目录结构**

| **层级**     | **一级目录**                        | **对外价值**                                | **是否需要独立 UI**        |
|--------------|-------------------------------------|---------------------------------------------|----------------------------|
| **体验产品** | HuntAI Workmate                     | 统一工作入口、任务协同与跨域委托            | 是，主入口                 |
| **领域系统** | HuntAI Test                         | 质量事实、证据、门禁与放行                  | 是，工作区                 |
| **领域系统** | HuntAI SRE                          | 事故事实、调查、协作与复盘                  | 是，工作区                 |
| **共享能力** | HuntAI Knowledge                    | 企业知识的准确检索与引用                    | 轻量管理 UI                |
| **共享能力** | HuntAI Data                         | 受控、只读、可审计的数据查询                | 轻量工作区                 |
| **治理**     | HuntAI Admin                        | 身份、权限、策略、审计和租户治理            | 是，管理控制台             |
| **平台底座** | HuntAI Platform Core                | Runtime、Event、Memory、Scheduler、Registry | 通常不作为独立用户 App     |
| **生态扩展** | Skills / Connectors / MCP Interface | 标准化工作流与企业系统集成                  | 目录/配置 UI，而非业务 App |

## **为什么做这次收敛**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>避免“一个能力一个 App”</strong></p>
<p>Knowledge、Data 后端复杂，但主要服务 Workmate / Test / SRE，因此默认采用 backend-heavy、UI-light 的产品策略。</p></th>
<th><p><strong>保护领域权威</strong></p>
<p>TestRun、Gate、Incident 等权威状态由对应领域系统拥有。Workmate 可以解释、查询、委托，但不越权成为状态机。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>把集成做完整</strong></p>
<p>Connector 负责认证、权限映射、Webhook、同步、重试与审计；MCP 只是 Agent-facing 的适配接口之一。</p></td>
<td><p><strong>为主动式 Agent 留接口</strong></p>
<p>Event System 与 Scheduler 是平台底座，使 HuntAI 从“问一句做一次”演进到“系统事件触发、Agent 判断并执行”。</p></td>
</tr>
</tbody>
</table>

**02 / PORTFOLIO MAP**

# **产品全景图**

对外产品目录建议保持少而清晰；对内架构则保留更细的 Runtime、Registry、Policy、Event 等基础设施。两者不是一回事，硬把微服务名塞进产品页只会制造会议。

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>DEFAULT EXPERIENCE</strong></th>
<th><p><strong>HuntAI Workmate</strong></p>
<p>统一入口 · General Work Agent · Task / Reminder · Delegation</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>DOMAIN SYSTEMS</strong></th>
<th><p><strong>HuntAI Test | HuntAI SRE</strong></p>
<p>质量事实与放行 生产事故事实与调查</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>SHARED CAPABILITIES</strong></th>
<th><p><strong>HuntAI Knowledge | HuntAI Data</strong></p>
<p>知识检索与引用 受控数据查询</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>GOVERNANCE</strong></th>
<th><p><strong>HuntAI Admin</strong></p>
<p>Identity · RBAC · Policy · Audit · Tenant / Scope</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>PLATFORM CORE</strong></th>
<th><p><strong>Agent Runtime · Event · Memory · Scheduler · Registries</strong></p>
<p>统一运行时与跨产品底座</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>INTEGRATION ECOSYSTEM</strong></th>
<th><p><strong>Skills · Connectors · MCP Interface</strong></p>
<p>Jira · Confluence · Git · CI/CD · K8s · Monitoring · Database</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## **核心职责边界**

| **对象**           | **回答的问题**           | **应该拥有**                             | **不应该承担**           |
|--------------------|--------------------------|------------------------------------------|--------------------------|
| **Agent**          | 谁来判断下一步做什么？   | 理解目标、规划、选择 Skill、委托、验证   | 替代权限系统和领域状态机 |
| **Skill**          | 重复任务如何标准化执行？ | 版本化流程、输入输出、步骤与验证         | 长期拥有业务权威状态     |
| **MCP / API**      | Agent 能访问和执行什么？ | Agent-facing 能力契约                    | 认证同步等完整集成职责   |
| **Connector**      | 如何可靠连接企业系统？   | 认证、API、Webhook、Sync、权限映射、审计 | 复杂业务判断             |
| **UI / Workspace** | 人在哪里观察和协作？     | 复杂状态、证据、审批、协作               | 为了“有后端”就强行造 App |

## 产品归类判定

| **体验产品**   | 面向广泛用户提供统一入口、个人上下文与跨域委托；不替代专业领域的权威状态。                |
|----------------|-------------------------------------------------------------------------------------------|
| **领域系统**   | 存在独立权威对象、状态机、审批或多人协作时，作为 Domain System，并提供专用 Workspace。    |
| **共享能力**   | 被多个 Agent / Domain System 复用、后端能力重但用户入口可轻时，作为 Platform Capability。 |
| **连接与接口** | Connector 负责完整集成生命周期；MCP / API 负责把受控能力暴露给 Agent。                    |

**EXPERIENCE PRODUCT**

**HuntAI Workmate**

| **公司的默认 AI 工作入口：找信息、办事情、跟进任务，并把专业问题委托给正确的领域系统。** |
|------------------------------------------------------------------------------------------|

| **主要用户** | 所有知识工作者、研发、测试、SRE、项目与管理人员。                                       |
|--------------|-----------------------------------------------------------------------------------------|
| **核心角色** | UI Shell + General Work Agent + Personal Context + Task / Reminder + Delegation。       |
| **权威对象** | 个人任务、提醒、计划任务、个人/项目上下文、委托记录；不拥有 TestRun 或 Incident 终态。  |
| **交互形态** | Chat / Today / Tasks 为主，辅以 Knowledge Search、Data Query Preview、Test / SRE 深链。 |
| **价值主张** | 一个入口完成日常工作，并在复杂领域问题上自动“找对系统、带回结果”。                      |

## **核心能力**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Daily Work</strong></p>
<p>今日任务、阻塞提示、Jira / PR / Meeting / Email 摘要、个人 Todo。</p></th>
<th><p><strong>Ask &amp; Act</strong></p>
<p>知识查询、数据查询、调用 Skills、执行受控工具动作。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Scheduler</strong></p>
<p>提醒、定时工作、周期检查和事件后的后续动作。</p></td>
<td><p><strong>Delegation</strong></p>
<p>识别测试/事故问题，委托 Test 或 SRE Agent，汇总结果并提供深链。</p></td>
</tr>
</tbody>
</table>

## **典型场景**

- “今天有哪些 PR / Jira / meeting 需要我处理？” → 汇总工作上下文并生成 Today 视图。

- “查一下支付接口的设计约束。” → 调用 Knowledge，返回答案、引用和来源新鲜度。

- “昨天订单失败率是多少？” → 调用 Data，展示 SQL Preview 和只读结果。

- “调查 payment service 上午的异常。” → 识别为 SRE 问题，创建/进入 Investigation 并在 Workmate 回报。

|     | **边界** Workmate 的产品边界要守住：它负责“找人和办事”，不是把 Test、SRE 的状态机搬进聊天框。跨域修改必须走对应系统的正式 Command / Policy。 |
|-----|----------------------------------------------------------------------------------------------------------------------------------------------|

**DOMAIN SYSTEM**

**HuntAI Test**

| **质量事实源：把 regression、evidence、gate、approval 和 release readiness 组织成可审计的放行系统。** |
|-------------------------------------------------------------------------------------------------------|

| **主要用户** | QA、研发、Release Owner、审批人、项目负责人。                                             |
|--------------|-------------------------------------------------------------------------------------------|
| **权威对象** | TestRun、Case / Execution、Evidence、GateEvaluation、ApprovalRequest、Release Readiness。 |
| **主界面**   | Test Run Workspace，而不是 Chat；Chat 用于解释失败和辅助调查。                            |
| **关键原则** | Agent Mode 的产出不能直接进入门禁；门禁、审批和状态变化必须经过正式命令与策略。           |
| **系统关系** | 可读取 SRE 的生产状态；可被 Workmate 查询/解释/深链，但不由 Workmate 拥有。               |

## **核心能力**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Execution &amp; Regression</strong></p>
<p>创建、运行、重跑 regression；聚合 case、环境、日志与失败信息。</p></th>
<th><p><strong>Evidence</strong></p>
<p>收集截图、日志、报告、构建产物与验证证据，形成可追踪 Evidence。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Gate &amp; Approval</strong></p>
<p>基于规则评估 Gate，发起 Approval Request，保留决策与审计链。</p></td>
<td><p><strong>AI Assistance</strong></p>
<p>失败归因、相似问题检索、证据摘要、release preparation，但不绕过治理。</p></td>
</tr>
</tbody>
</table>

## **推荐工作区结构**

| **区域**              | **内容**                                   |
|-----------------------|--------------------------------------------|
| **Test Run**          | Cases · Execution · Failures · Environment |
| **Evidence**          | Logs · Screenshots · Reports · Attachments |
| **Gate**              | 规则命中 · 风险解释 · 阻塞项               |
| **Approval**          | 审批人 · 决策 · 备注 · 审计记录            |
| **Release Readiness** | 版本整体状态 · 关键风险 · 待处理事项       |

**DOMAIN SYSTEM**

**HuntAI SRE**

| **生产事故事实源：围绕 Incident 聚合告警、时间线、观测数据、变更与调查结论。** |
|--------------------------------------------------------------------------------|

| **主要用户** | SRE、On-call 工程师、服务 Owner、Incident Commander、研发值班人员。                  |
|--------------|--------------------------------------------------------------------------------------|
| **权威对象** | Incident、Timeline、Investigation、Alert、On-call、Postmortem。                      |
| **主界面**   | Incident / Investigation Workspace；Chat 作为调查助手，而不是事故系统本身。          |
| **数据边界** | Metrics / Logs / Traces / K8s Events 属于 Observability；不能用 NL2SQL / Data 替代。 |
| **跨域关系** | 可读取 Test 的 TestRun / Release 事实，但不直接修改 Test 权威状态。                  |

## **Investigation Workspace**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Situation</strong></p>
<p>Incident 概况、Severity、影响范围、Owner、当前状态。</p></th>
<th><p><strong>Timeline</strong></p>
<p>告警、部署、配置变更、人工操作与关键发现的统一时间轴。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Observability</strong></p>
<p>Metrics、Logs、Traces、K8s Events、异常关联。</p></td>
<td><p><strong>Changes</strong></p>
<p>近期 Deployment、PR、Feature Flag、依赖变化。</p></td>
</tr>
<tr class="even">
<td><p><strong>Knowledge</strong></p>
<p>Runbook、Architecture、历史 Incident、变更文档的定向检索。</p></td>
<td><p><strong>Postmortem</strong></p>
<p>调查结论、Root Cause、Action Items、复盘材料与知识沉淀。</p></td>
</tr>
</tbody>
</table>

|     | **关键边界** SRE 与 Data 的差异必须在目录里讲清楚：Data 回答业务/数仓问题；SRE Observability 回答“系统现在为什么异常”。两者底层数据、权限和时效性都不同。 |
|-----|-----------------------------------------------------------------------------------------------------------------------------------------------------------|

**SHARED CAPABILITY**

**HuntAI Knowledge**

| **企业知识检索平台：让 Workmate、Test、SRE 能够基于正确来源、正确权限、正确引用回答问题。** |
|---------------------------------------------------------------------------------------------|

| **主要客户**   | Workmate、SRE、Test 等 Agent / Domain System；知识管理员通过轻量 UI 管理来源。                             |
|----------------|------------------------------------------------------------------------------------------------------------|
| **定位**       | Enterprise Knowledge Retrieval Platform，不是另一个 Docs / Confluence。                                    |
| **核心职责**   | Source、Sync、ACL、Classification、Chunking、Indexing、Hybrid Retrieval、Reranking、Citation、Evaluation。 |
| **管理界面**   | Source Management、Scope、ACL、Sync Health、Evaluation；不重做编辑器、目录树和评论系统。                   |
| **准确性原则** | 源对、新鲜、权限正确、混合检索 + 重排、强制引用、黄金问题集评测。                                          |

## **两层产品面**

| **产品面**                      | **职责**                                  | **主要用户**                          |
|---------------------------------|-------------------------------------------|---------------------------------------|
| **Knowledge Source Management** | 来源、范围、分级、ACL、同步健康、索引策略 | Admin / Knowledge Owner               |
| **Knowledge Retrieval**         | 检索、重排、引用、答案上下文、评测接口    | Workmate / Test / SRE / Future Agents |

## **优先场景**

- SRE：检索 Runbook、Architecture、历史 Incident 和 Deployment Knowledge，并强制附原始引用。

- Test：检索测试规范、release checklist、已知缺陷与历史证据。

- Workmate：回答企业政策、项目文档、研发流程与团队知识问题。

|     | **产品策略** 第一批重点客户不必只是 Workmate。SRE 对“正确来源 + 高新鲜度 + 强引用”的需求更刚性，可以反向推动 Knowledge 的质量体系成熟。 |
|-----|-----------------------------------------------------------------------------------------------------------------------------------------|

**SHARED CAPABILITY**

**HuntAI Data**

| **受控、只读、可审计的数据查询能力：自然语言只是入口，治理和执行安全才是产品核心。** |
|--------------------------------------------------------------------------------------|

| **主要客户** | Workmate、Test、SRE 及有数据查询需求的业务/研发用户。                                                             |
|--------------|-------------------------------------------------------------------------------------------------------------------|
| **定位**     | Enterprise Data Query Capability / Controlled Read-only Query Gateway；NL2SQL 只是能力之一。                      |
| **核心对象** | Connections、Data Catalog、Schema、Permission、Query History、Audit。                                             |
| **执行链路** | Authorization → Schema Discovery → SQL Generation → Validation → Policy → Preview → Read-only Execution → Audit。 |
| **关键原则** | 权限不能依赖 Prompt；默认只读；高风险数据与大查询受策略约束。                                                     |

## **用户体验**

| **入口**           | **体验**                                                 | **适用角色**               |
|--------------------|----------------------------------------------------------|----------------------------|
| **Workmate 内嵌**  | 自然语言问题 → SQL Card → SQL Preview → 只读结果         | 普通业务 / 研发用户        |
| **Data Workspace** | Catalog、Authorization、Query History、Connection Health | 数据 Owner / 管理员        |
| **API / MCP**      | 受控查询能力供其他 Agent 调用                            | Test / SRE / Future Agents |

|     | **产品策略** 不建议一开始把 Data 做成完整 BI 产品。只有当出现稳定的独立用户群、复杂领域对象和持续可视分析需求时，再考虑升格。 |
|-----|-------------------------------------------------------------------------------------------------------------------------------|

**GOVERNANCE**

**HuntAI Admin**

| **企业级治理控制台：把身份、权限、策略、审计、连接范围和产品配置从 Prompt 中拿出来。** |
|----------------------------------------------------------------------------------------|

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Identity &amp; RBAC</strong></p>
<p>SSO、用户/组、角色、Project / Tenant Scope。</p></th>
<th><p><strong>Policy</strong></p>
<p>工具权限、数据访问、状态变更、审批策略和风险控制。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Audit</strong></p>
<p>Agent、Skill、Connector、Data Query、状态变更的审计记录。</p></td>
<td><p><strong>Configuration</strong></p>
<p>Knowledge Sources、Data Connections、Connectors、Registry、Feature Scope。</p></td>
</tr>
</tbody>
</table>

## **HuntAI Platform Core**

Platform Core 是本版新增的“目录包装层”：它把原讨论中横向共享的基础设施归为一个统一底座。它不是新的用户工作台，而是所有产品共同依赖的运行与治理基础。

| **能力**                      | **职责**                                                      |
|-------------------------------|---------------------------------------------------------------|
| **Agent Runtime**             | 统一的执行、planning、tool calling、delegation 和验证运行时。 |
| **Event System**              | 接收外部系统事件并触发策略判断、Agent / Skill 执行。          |
| **Memory**                    | 受控保存个人/项目上下文，服务 Workmate 与 Agent Runtime。     |
| **Scheduler**                 | Reminder、定时任务、周期检查和事件后的延迟动作。              |
| **Skill Registry**            | Skill 版本、依赖、范围、发布和启停管理。                      |
| **MCP / Capability Registry** | 记录 Agent-facing 能力接口、版本、权限与可用性。              |
| **Identity / Policy / Audit** | 所有产品共享的企业治理基线。                                  |

|     | **目录原则** Memory、Scheduler、Registry 很重要，但不建议在面向客户的一级目录中各自包装成产品。否则产品线会迅速退化成“基础设施服务名词展览”。 |
|-----|-----------------------------------------------------------------------------------------------------------------------------------------------|

**ECOSYSTEM**

**Skills & Connectors**

| **Skills 把重复工作标准化；Connectors 把企业系统可靠接入；MCP / API 把这些能力暴露给 Agent。** |
|------------------------------------------------------------------------------------------------|

## **HuntAI Skills**

| **定义** | 标准化、版本化、可复用的工作流程，回答“How do I perform a repeatable task?”                                           |
|----------|-----------------------------------------------------------------------------------------------------------------------|
| **示例** | prepare-release、analyze-failed-regression、investigate-api-latency、prepare-jira-ticket、generate-release-evidence。 |
| **治理** | Install / Enable / Version / Project Scope / Permission / Audit。                                                     |
| **边界** | Skill 不等于 Agent；它不应自行成为长期权威状态的拥有者。                                                              |

## **HuntAI Connectors**

| **定义**       | 完整的企业系统集成层。                                                                                                        |
|----------------|-------------------------------------------------------------------------------------------------------------------------------|
| **职责**       | Authentication、Authorization、Credential、API Client、Webhook、Sync、Mapping、Rate Limit、Retry、Audit、Permission Mapping。 |
| **首批连接器** | Jira、Confluence、Git、CI/CD、Kubernetes、Monitoring、Database。                                                              |
| **MCP 关系**   | MCP 是可能的 Agent-facing Adapter；Connector 才负责完整集成生命周期。                                                         |

## **推荐 Connector 结构**

| **Connector**            | **内部能力**                                             | **Agent-facing**                |
|--------------------------|----------------------------------------------------------|---------------------------------|
| **Jira Connector**       | API Client · Webhook · Sync · Permission Mapping · Audit | MCP / API Adapter               |
| **Confluence Connector** | Auth · Content Sync · ACL Mapping · Webhook              | Knowledge Ingestion + MCP / API |
| **Git / CI Connector**   | PR / Commit / Build / Artifact / Event                   | Test / SRE / Workmate Tools     |
| **Monitoring Connector** | Alert · Metrics/Logs/Traces Reference · Event            | SRE Tools / Event Trigger       |
| **Database Connector**   | Connection · Schema · Permission · Read-only Execution   | Data Capability                 |

**03 / SOLUTION PACKAGES**

# **产品组合：从“模块”变成“解决方案”**

客户真正关心的通常不是“你们有几个 Agent”，而是“这套东西能解决什么工作问题”。因此目录应同时提供产品视角和场景组合视角。

| **解决方案**                  | **核心组合**                                                       | **典型价值**                                                            |
|-------------------------------|--------------------------------------------------------------------|-------------------------------------------------------------------------|
| **AI Work Hub**               | Workmate + Knowledge + Skills + Jira/Confluence/Git Connectors     | 统一入口完成日常工作、知识查询、任务跟进与轻量自动化。                  |
| **Release Quality**           | Test + Workmate + Knowledge + Git/CI/Jira Connectors               | 从 regression 到 evidence、gate、approval 和 release readiness 的闭环。 |
| **Production Reliability**    | SRE + Knowledge + Monitoring/K8s/Git Connectors + Test Read Access | 从 P1 告警到调查、时间线、变更关联和复盘。                              |
| **Controlled Data Assistant** | Data + Workmate + Database Connectors + Admin                      | 自然语言查询 + SQL Preview + 只读执行 + 权限与审计。                    |

## **跨产品协作示例**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Release 失败</strong></p>
<p>CI BuildFailed → Test 创建/更新 TestRun → Agent 分析失败 → Knowledge 检索规范 → Workmate 通知负责人。</p></th>
<th><p><strong>线上事故</strong></p>
<p>Monitoring P1Alert → SRE Incident → 聚合观测数据和近期部署 → 引用 Test 最近 release 事实 → Workmate 汇报。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>数据问答</strong></p>
<p>Workmate 接收业务问题 → Data 授权和 schema discovery → SQL Preview → 只读执行 → Audit。</p></td>
<td><p><strong>知识更新</strong></p>
<p>Confluence Webhook → Connector 同步 → Knowledge 更新索引/ACL → 后续 Agent 检索返回最新引用。</p></td>
</tr>
</tbody>
</table>

**04 / AUTOMATION MODEL**

# **主动式工作：Event-driven 是平台能力，不是额外产品**

只有“Agent → MCP → Tool”还不够。企业工作大量由系统事件驱动：构建失败、Jira 更新、P1 告警、回归失败。Event System 让 HuntAI 从被动问答演进为主动发现与执行。

| **事件源**           | **事件**         | **目标产品** | **典型动作**                             |
|----------------------|------------------|--------------|------------------------------------------|
| **Jira**             | IssueUpdated     | Workmate     | 提醒、重新检查阻塞、触发项目 Skill。     |
| **CI/CD**            | BuildFailed      | Test         | 更新 TestRun、抓取证据、分析失败。       |
| **Monitoring**       | P1Alert          | SRE          | 创建/关联 Incident、启动 Investigation。 |
| **Test**             | RegressionFailed | Workmate     | 通知 Owner、生成待办、提供 Test 深链。   |
| **Knowledge Source** | ContentChanged   | Knowledge    | 增量同步、ACL 更新、索引刷新。           |

|     | **治理底线** 事件触发不等于“Agent 自动获得无限权限”。每次动作仍需经过身份、策略、风险等级、审批和审计链。 |
|-----|-----------------------------------------------------------------------------------------------------------|

## **统一执行模型**

| **步骤**       | **说明**                                             |
|----------------|------------------------------------------------------|
| **1. Trigger** | User Request / Schedule / External Event。           |
| **2. Context** | 加载用户、项目、领域和权限上下文。                   |
| **3. Decide**  | Agent 判断任务类型，选择 Skill / Domain System。     |
| **4. Act**     | 通过 Connector + MCP / API 执行受控动作。            |
| **5. Verify**  | 校验结果、状态、证据与策略。                         |
| **6. Persist** | 写入对应权威系统；个人上下文进入 Workmate / Memory。 |
| **7. Notify**  | 在 Workmate / Test / SRE Workspace 或外部渠道回报。  |

**05 / OWNERSHIP MODEL**

# **权威对象与写入边界**

这是产品边界最容易在实现阶段被“顺手”破坏的地方。下面这张表应该进入架构评审和 API Review 的长期检查清单。

| **权威对象**                 | **Owner**    | **Workmate** | **Test** | **SRE** | **其他产品写入方式**        |
|------------------------------|--------------|--------------|----------|---------|-----------------------------|
| **Personal Task / Reminder** | Workmate     | 读写         | 读       | 读      | Workmate Command            |
| **TestRun / Evidence**       | Test         | 查询/解释    | 读写     | 读      | Test Command + Policy       |
| **Gate / Approval**          | Test         | 查询/草稿    | 读写     | 只读    | Test Approval API           |
| **Incident / Timeline**      | SRE          | 查询/委托    | 只读     | 读写    | SRE Command + Policy        |
| **Knowledge Index**          | Knowledge    | 调用         | 调用     | 调用    | Knowledge Ingestion / Admin |
| **Data Query / Audit**       | Data         | 调用         | 调用     | 调用    | Data Policy + Runtime       |
| **Identity / Policy**        | Admin / Core | 遵循         | 遵循     | 遵循    | Admin Only                  |

|     | **设计原则** 跨域读取可以丰富调查和判断；跨域写入必须显式、可审计、经策略控制。不要让 Agent 直接修改另一个领域数据库对象。 |
|-----|----------------------------------------------------------------------------------------------------------------------------|

## **UI 决策规则**

是否做独立 UI，不看“后端有多复杂”，看是否存在必须离开对话框才能完成的复杂状态、权威对象、工作流与多人协作。

| **模块**       | **Chat 价值** | **独立 UI 价值** | **推荐形态**                        |
|----------------|---------------|------------------|-------------------------------------|
| **Workmate**   | 极高          | 中               | Chat / Today / Tasks 主入口         |
| **Test**       | 高            | 极高             | 独立 Test Workspace                 |
| **SRE**        | 极高          | 极高             | 独立 Incident Workspace             |
| **Knowledge**  | 高            | 中               | 轻量 Source / Evaluation 管理 UI    |
| **Data**       | 高            | 中高             | 嵌入 Workmate + 轻量 Data Workspace |
| **Connectors** | 低            | 中               | Admin / Settings 配置 UI            |

**06 / ROADMAP**

# **产品演进路线**

路线沿用讨论中的优先级，但把“平台底座”和“产品化门槛”补齐：先建立质量事实与治理基线，再扩展默认入口和共享能力，最后把 SRE 与事件驱动做成完整闭环。

| **阶段**    | **重点**                          | **可交付结果**                                                 | **进入下一阶段的判断**     |
|-------------|-----------------------------------|----------------------------------------------------------------|----------------------------|
| **Phase 0** | Test Foundation + Core Governance | TestRun / Evidence / Gate 基础；Identity / Policy / Audit 基线 | 质量对象和权限边界稳定     |
| **Phase 1** | Workmate Lite                     | Chat / Today / Tasks；基础 Skills / Connectors；Test 深链      | 形成高频日常入口           |
| **Phase 2** | Knowledge + Data Capability       | 权威知识检索、引用；受控数据查询；轻量管理 UI                  | 共享能力被多个产品真实复用 |
| **Phase 3** | SRE                               | Incident / Timeline / Investigation；Monitoring / K8s 集成     | 形成独立生产事故事实源     |
| **Phase 4** | Platform Shell + Event-driven     | 统一 Delegation、Event、Scheduler、Registry 和跨产品体验       | Workmate 成为公司默认入口  |

## **每个产品的“升格条件”**

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Knowledge</strong></p>
<p>只有当独立知识运营、质量评测、来源治理形成稳定角色与流程时，才扩展更完整的独立工作台。</p></th>
<th><p><strong>Data</strong></p>
<p>只有当独立用户群、查询资产、分析工作流和可视化需求持续增长时，才考虑完整 BI 化。</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Skills</strong></p>
<p>优先作为 Registry / Catalog，不要过早做成高权限“技能商店”。</p></td>
<td><p><strong>Future Agents</strong></p>
<p>新 Agent 必须先回答：是否拥有独立领域事实？若没有，优先做 Skill / Capability，而不是新产品。</p></td>
</tr>
</tbody>
</table>

**07 / FINAL CATALOG**

# **最终对外产品目录建议**

| **产品**                 | **一句话定义**                                               | **产品形态**                  | **主要依赖**                                  |
|--------------------------|--------------------------------------------------------------|-------------------------------|-----------------------------------------------|
| **HuntAI Workmate**      | 负责找信息、办事情、跟任务，并把专业问题委托给正确系统。     | 体验产品 / General Work Agent | Core · Skills · Connectors · Knowledge / Data |
| **HuntAI Test**          | 负责质量事实、测试证据、门禁、审批与放行。                   | 领域系统 / Quality System     | Core · CI/Git/Jira · Knowledge                |
| **HuntAI SRE**           | 负责生产事故、调查、时间线、协作与复盘。                     | 领域系统 / Incident System    | Core · Monitoring/K8s/Git · Knowledge         |
| **HuntAI Knowledge**     | 负责企业知识的准确检索、权限继承与引用。                     | 共享能力 / Retrieval Platform | Connectors · Admin · Core                     |
| **HuntAI Data**          | 负责受控、只读、可审计的数据查询。                           | 共享能力 / Query Gateway      | DB Connectors · Admin · Core                  |
| **HuntAI Admin**         | 负责身份、权限、策略、审计与企业级配置。                     | 治理控制台                    | Platform Core                                 |
| **HuntAI Platform Core** | 负责统一 Runtime、Event、Memory、Scheduler 与 Registries。   | 平台底座                      | Identity · Policy · Infrastructure            |
| **Skills & Connectors**  | 负责流程复用和企业系统集成；MCP/API 暴露 Agent-facing 能力。 | 生态扩展层                    | Admin · Core                                  |

## **命名建议**

| **原名称 / 叫法**  | **建议名称**                                  | **原因**                                            |
|--------------------|-----------------------------------------------|-----------------------------------------------------|
| **huntai-docs**    | HuntAI Knowledge                              | 避免让人误以为要重做文档编辑/协作系统。             |
| **huntai-nl2sql**  | HuntAI Data                                   | NL2SQL 只是入口之一，产品核心是受控查询和治理。     |
| **Jira MCP**       | Jira Connector + MCP Adapter                  | 把完整集成职责和 Agent-facing 接口拆清。            |
| **五个并列 Agent** | Experience + Domain + Capability + Governance | 按事实所有权和用户工作流划分，而不是按 Agent 数量。 |

|     | **HuntAI 产品生态** 一句话总结：Workmate 是默认体验；Test / SRE 是权威领域系统；Knowledge / Data 是共享能力；Admin / Platform Core 提供治理与运行底座；Skills / Connectors 负责复用与连接。 |
|-----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## **明确不做**

- 不为每个后端能力单独做一个完整 App。

- 不把 Test / SRE 降级成普通 Skill，也不让 Workmate 拥有其终态。

- 不重做 Confluence，不把 Data 直接做成完整 BI。

- 不让 Prompt 决定权限，不让 Agent 绕过审批、状态机和领域命令。

- 不把个人 token、开放高权限技能商店、跨域直写数据库作为捷径。

- 不使用 NL2SQL 替代 SRE 的 Metrics / Logs / Traces / K8s Observability。

**APPENDIX**

# **附录：产品卡片模板**

后续新增任何产品、Agent 或 Capability，都建议先用这张模板回答清楚边界，再决定它应该进入产品目录、Skill Registry 还是 Connector Catalog。

| **问题**                    | **需要回答的内容**                                       |
|-----------------------------|----------------------------------------------------------|
| **用户是谁？**              | 是否存在稳定、可识别的用户群与高频工作任务？             |
| **它拥有何种事实？**        | 是否有独立的权威对象、状态机和审计责任？                 |
| **核心价值是什么？**        | 一句话能否描述业务结果，而不是技术实现？                 |
| **需要独立 UI 吗？**        | 是否存在复杂状态、证据、审批或多人协作？                 |
| **它是 Agent 还是 Skill？** | 是否需要自主规划与委托，还是可复用流程即可？             |
| **它如何连接外部系统？**    | Connector 的认证、同步、Webhook、权限映射如何设计？      |
| **写入边界是什么？**        | 哪些对象可直接写，哪些必须通过 Domain Command / Policy？ |
| **如何被其他产品复用？**    | API / MCP / Event / Deep Link / Read Model 分别是什么？  |

## **编制说明**

本目录以用户提供的《HuntAI 开发套件生态设计讨论整理》为基础，并在不改变其核心职责边界的前提下，增加了面向“产品目录”的包装建议：引入 Platform Core 作为底座聚合项；将 Memory / Scheduler 从一级产品下沉；将 Skills / Connectors 归入生态扩展层；增加解决方案组合、权威对象矩阵和产品升格条件。
