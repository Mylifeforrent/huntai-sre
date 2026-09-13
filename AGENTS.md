# AGENTS.md — huntai-sre 仓库级指令（Stage 2 初始版）

> Cursor / ZCode 读取本文。Claude Code 入口为根 `CLAUDE.md`（指向本文，禁止另写规则）。
> 跨项目偏好见个人级 `~/.zcode/AGENTS.md`。工程细则见 [`docs/00_setup/project_rules.md`](docs/00_setup/project_rules.md)。
> 本阶段只写工程规则。设计资产约束由 Stage 14 补进本文；在此之前禁止把未产出的业务规则写成事实。

**新会话起步**：先读本文 → `project_rules.md` → 按下表打开对应阶段文档。文档不存在 ⇒ 该阶段未完成，禁止发明其内容。

| 要做什么 | 先打开 |
|---|---|
| 命名 / Git / 验证命令 / 红线 | `docs/00_setup/project_rules.md` |
| 目录职责与阶段产出 | 本文 §1–§4；各 `docs/NN_*/README.md` |
| 技术栈推荐（未冻结） | 根 `README.md`「技术栈决策」 |
| 改已有结论或冻结资产 | 先写 `docs/13_changes/change_log.md` |

## 1. 目录职责边界

```text
frontend/   前端源码与前端工程配置。禁止放设计文档或后端代码。
backend/    后端源码与后端工程配置。禁止放设计文档或前端代码。
docs/       全部设计资产与过程文档的唯一存放地。禁止放可执行代码（Markdown 内示例除外）。
```

仓库顶层只允许上述三个业务目录，以及仓库级配置：`README.md`、`AGENTS.md`、`CLAUDE.md`、`.gitignore`、`.env.example`、`.pre-commit-config.yaml`、`.gitleaks.toml`、`.cursor/`、`.mcp.json`。Stage 12 才允许新增顶层 `docker-compose.yml` 与 `.github/`；二者只放编排与 CI 门控，不得写入业务规则。CI 门控命令必须与 `project_rules.md` §5 一致。

- `docs/` 阶段目录名 `00_setup`–`13_changes`：序号与名称禁止变更。
- 阶段产物只进对应 `docs/NN_*`。约定输出文件不存在 ⇒ 该阶段未完成，禁止进入下一阶段（`13_changes` 除外，自本阶段起持续更新）。
- `backend/` / `frontend/` 子目录按 Stage 6 已定稿架构落地。现阶段只允许各一份 `README.md`。禁止预建设计未规定的模块目录，禁止在本阶段生成实现代码。
- 设计文档写进代码目录，或代码写进 `docs/`，当次提交必须移正。

## 2. 配置、依赖与虚拟环境

- 配置唯一入口：仓库根 `.env`（禁止 `backend/.env` / `frontend/.env` 另立一份）。代码禁止硬编码配置值。
- `.env` 不入库。`.env.example` 只含键名与注释，禁止填写真实值。新增配置项必须同步 `.env.example`。
- 后端依赖与虚拟环境：必须用 **uv**（`pyproject.toml` + `uv.lock`）。禁止 pip 手写 `requirements.txt`、禁止 poetry。锁文件变更必须独立成 commit。
- 前端依赖：必须用 **npm** + `package-lock.json`。禁止 pnpm / yarn。
- 运行时依赖新增顺序：用户批准 → 审计（维护活跃度 / License / 无未修复通告）→ `uv add` 或 `npm install` 落锁。禁止 AI 自行引入。
- 密钥、生产数据、真实 PII 禁止写入仓库、日志、示例、Prompt。需要 AI 协助时只用脱敏或合成样例。

版本锁定与「不采纳清单」在 Stage 6 产出后以该阶段文档为准；在此之前以根 `README.md` 的推荐组合为工作假设，并逐条保留 `[ASSUMPTION]` / `[TBD-*]`。

## 3. 阶段约定输出（路径 + 格式）

每个 Agent Team 阶段必须落盘下表文件。缺文件 = 阶段未完成。文件名相对仓库根。格式除注明外均为 Markdown。

| 阶段目录 | 产出阶段 | 约定输出 |
|---|---|---|
| `docs/00_setup/` | Stage 2 初始化 | `project_rules.md`；根 `AGENTS.md`（本文） |
| `docs/01_market_research/` | 业务调研 | `market_research_report.md` |
| `docs/02_competitor_analysis/` | 竞品分析 | `competitor_analysis_report.md` |
| `docs/03_problem_modeling/` | 问题建模 | `business_model.md` |
| `docs/04_interaction_design/` | 交互链路 | `interaction_design_summary.md` |
| `docs/05_prototype/` | 原型 | `prototype_link.md`；`prototype_review.md`（Gate 1） |
| `docs/06_architecture_design/` | 系统架构 | `architecture_spec.md`；`frontend_design_spec-v1.0.md`；`frontend_backend_boundary_spec-v1.0.md` |
| `docs/07_backend_design/` | 数据与 API | `data_model_spec.md`；`api_interface_spec.md`；`gate2_review.md`（Gate 2） |
| `docs/08_prd/` | PRD | `prd_document.md` |
| `docs/09_figma_highfi/` | 高保真 | `design_tokens.json`；`component_spec.json` |
| `docs/10_ai_context/` | AI 上下文 | `ai_context_index.md` |
| `docs/11_test/` | 实现与联调 | `test_plan.md`；`code_review_report.md`（Gate 3）；`e2e_report.md` |
| `docs/12_deployment/` | 发布部署 | `cicd_pipeline_spec.md`；`release_checklist.md`；`gate4_review.md`（Gate 4） |
| `docs/13_changes/` | 变更（持续） | `change_log.md` |

手册 0.8 将 `architecture_spec.md` 列在 `07_backend_design/`。本仓库按目录职责把架构放在 `06_architecture_design/`，数据模型与 API 放在 `07_backend_design/`。禁止再按 0.8 文件表把架构正文写入 07。

研究材料若在仓库外（如 `opensource-product-analysis`），只允许路径引用，禁止把外部研究包正文复制进本仓库冒充本阶段产出。

## 4. 复杂任务必须先 Plan 再实施

满足任一条件必须先输出 Plan（目标、涉及文件、步骤、验证方式），经用户确认再改文件：

1. 两个及以上文件；
2. 修改已有代码行为或已有文档结论；
3. 新增目录或顶层文件；
4. 变更依赖或配置。

仅单文件文案/注释、纯格式化、新建空目录可直接做。

实现类任务一个模块一个新会话，以 Task Brief 开场（模板见流程手册附录 E；未入库前以会话内同等六段式为准）。单次会话引用文档 ≤ 5 份，其余用路径引用。

## 5. 权威、标记与冻结

输入权威优先级：用户当次指令 → 本文 → `docs/00_setup/project_rules.md` → 已产出的 `docs/NN_*` 约定文件。工程过程（命名 / Git / 验证 / 红线）上 `project_rules.md` 优于本文。

- 缺失输入标记 `[ASSUMPTION]` 并写明假设，不得当作既定事实。
- 规范未定标记 `[TBD-BIZ]` / `[TBD-UX]` / `[TBD-FE]` / `[TBD-BE]` / `[TBD-INFRA]`，写明由仓库维护人在何阶段确认。禁止用假默认值填满文档。
- 两份已产出文档冲突：标 `[CONFLICT]`、写入 `change_log.md`、fail-close（拒绝动作、不发明折中），等待用户裁决。

**冻结资产**（改前必须：change_log 登记 + 用户显式批准 + 回复中展示 diff）：

1. 文首 Status 为「已冻结」或「已定稿」的文档；
2. 已被后续阶段消费的设计产出（即使 Status=Draft）；
3. `project_rules.md`、本文、`CLAUDE.md`、`.env.example`、已存在的 MCP 清单（`.mcp.json`、`.cursor/mcp.json`）。

## 6. 提交与验证

- 代码变更走 `feat/<slug>` 或 `fix/<slug>`，squash 回 `main` 后删分支。文档与规范可直接提交 `main`。
- Commit：`<type>(<scope>): <subject>`，规则见 `project_rules.md` §3。
- 禁止 `git commit --no-verify`。pre-commit（gitleaks）必须通过。
- AI 生成代码合入前：实际运行 `project_rules.md` §5 命令且零失败；逐项核对 §4 Checklist。任一项为否禁止合入。
- Stage 11 之前：禁止新增 `frontend/`、`backend/` 下的实现文件（各目录 `README.md` 除外）。合入前用 `git status` 确认无 `.py` / `.ts` / `.tsx` / `.js` / `.jsx` 等源码文件。

## 7. 禁止行为（Stage 2 可判定）

违反红线：立即停止，回滚本次变更；已入 `main` 则 revert 并登记 `change_log.md`。细则 `project_rules.md` §6。

1. 禁止将密钥、`.env` 值、生产数据、真实 PII 写入仓库、日志、示例、Prompt 或提供给 AI。
2. 禁止在未登记 change_log、未获用户显式批准、未展示 diff 的情况下修改冻结资产。
3. 禁止提交未实际运行验证且零失败的 AI 生成代码。
4. 禁止未经「用户批准 → 审计 → 落锁」引入运行时依赖。
5. 禁止使用 pip / 手写 `requirements.txt` / poetry / pnpm / yarn。
6. 禁止硬编码配置值；禁止提交 `.env`；禁止 `.env.example` 填写真实值。
7. 禁止改 `docs/` 阶段目录序号或名称；禁止跳过约定输出文件进入下一阶段。
8. 禁止把 `[ASSUMPTION]` / `[TBD-*]` / 仓库外研究报告写成已批准产品结论。
9. 禁止在 Stage 11 之前生成前后端实现代码。
10. 禁止只改 MCP 清单中的一份（现阶段：`.mcp.json` 与 `.cursor/mcp.json` 必须一致）。

Stage 14 将根据已冻结的 PRD / 架构 / API / 数据模型，把领域对象、鉴权、错误码、页面与 Agent 边界补进本文。在那些文档产出之前，禁止在实现中发明上述规则。
