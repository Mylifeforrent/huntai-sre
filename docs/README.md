# docs — 设计资产索引

全部设计资产与过程文档的唯一存放地。可执行代码禁止进入本目录（Markdown 内示例除外）。
分层与流程手册 0.8 节目录一致；架构正文按本仓库目录职责放在 `06_architecture_design/`，不放在 `07_backend_design/`（见根 `AGENTS.md` §3）。

由各阶段按序产出；约定输出文件见子目录 `README.md`。缺文件 = 该阶段未完成。

| 目录 | 存放什么 | 产出阶段 |
|---|---|---|
| [00_setup](00_setup/) | 工程规范与 AI 红线 | Stage 2 初始化 |
| [01_market_research](01_market_research/) | 业务调研报告 | 业务调研 |
| [02_competitor_analysis](02_competitor_analysis/) | 竞品分析报告 | 竞品分析 |
| [03_problem_modeling](03_problem_modeling/) | 业务问题模型（含 NFR） | 问题建模 |
| [04_interaction_design](04_interaction_design/) | 核心交互链路 | 交互设计 |
| [05_prototype](05_prototype/) | 原型链接与 Gate 1 记录 | 原型 |
| [06_architecture_design](06_architecture_design/) | 系统架构与前后端边界 | 架构设计 |
| [07_backend_design](07_backend_design/) | 数据模型、API 契约、Gate 2 | 数据与 API |
| [08_prd](08_prd/) | PRD（含可测试 AC） | PRD |
| [09_figma_highfi](09_figma_highfi/) | Design Tokens 与组件规格 | 高保真 |
| [10_ai_context](10_ai_context/) | 供给 Agent 的精简上下文 | AI 上下文 |
| [11_test](11_test/) | 测试计划、Gate 3、E2E 报告 | 实现与联调 |
| [12_deployment](12_deployment/) | CI/CD、发布清单、Gate 4 | 发布部署 |
| [13_changes](13_changes/) | 变更登记 | 持续 |

仓库外研究输入（禁止冒充本仓库阶段产出）：`opensource-product-analysis/competitors/sre-agent-analysis/`。
