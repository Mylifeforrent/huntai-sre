# 项目工程规范（project_rules）

- **适用范围**：huntai-sre 全仓库（`frontend/`、`backend/`、`docs/`）
- **产出阶段**：Stage 2（项目初始化与规范搭建）
- **生效方式**：仓库根 `AGENTS.md`（初始版）为跨工具 Agent 指令；`CLAUDE.md` 指向 `AGENTS.md`，不得另写规则。本文件管工程过程（命名 / Git / 验证 / 红线 R1–R4）；与 `AGENTS.md` 在这些主题上冲突时，以本文件为准。业务对象、API、页面、技术栈版本在对应 `docs/` 阶段产出并冻结前禁止用本文件补全。
- **修订方式**：任何修订必须先在 `docs/13_changes/change_log.md` 登记，再以 `docs(setup):` 类型的 commit 合入
- **约束等级用语**：**必须**（违反即阻断合入）／ **禁止**（违反即触犯红线，处置见 §6）／ **默认**（不声明变更即按此执行）

## 1. 代码命名规范

### 1.1 后端（Python）

| 对象 | 规范 | 示例 |
|---|---|---|
| 模块 / 文件 | snake_case | `alert_service.py` |
| 类 | PascalCase | `AlertFingerprint` |
| 函数 / 方法 / 变量 | snake_case | `build_citation` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY = 3` |
| 环境变量 | UPPER_SNAKE_CASE | `POSTGRES_PASSWORD` |
| 包名 | 单个小写单词 | `app.api` |

- 公开函数与类方法必须带完整类型注解（参数 + 返回值）。
- **判定标准**：Stage 11 起 `uv run ruff check .` 与 `uv run ruff format --check .` 零告警；抽查公开函数无缺失注解。Stage 11 之前本条适用于「未引入 `.py` 实现文件」。

### 1.2 前端（TypeScript / React）

| 对象 | 规范 | 示例 |
|---|---|---|
| 组件文件 | PascalCase.tsx | `AlertTimeline.tsx` |
| Hook 文件 | use 前缀 + camelCase.ts | `useAlertStream.ts` |
| 工具 / 纯函数文件 | camelCase.ts | `formatTime.ts` |
| 组件名 / 类型 / 接口 | PascalCase | `CitedFinding` |
| 变量 / 函数 | camelCase | `submitApproval` |
| 常量 | UPPER_SNAKE_CASE | `MAX_EVENTS` |
| 自定义 CSS 类 | kebab-case（优先使用 Tailwind 原子类） | `alert-panel` |

- tsconfig 必须开启 `strict: true`；禁止 `any`，确需宽类型用 `unknown` + 类型收窄。
- **判定标准**：Stage 11 起 `npx tsc --noEmit` 与 ESLint 零错误；`rg ': any|<any>' frontend/src` 零命中。Stage 11 之前本条适用于「未引入 `.ts` / `.tsx` 实现文件」。

### 1.3 文档、数据与 API

- `docs/` 内文件名：全小写 snake_case，扩展名 `.md`（如 `market_research_report.md`）。JSON 资产用 snake_case 文件名。
- `docs/` 阶段目录名固定为 `NN_名称`（00–13）：序号与名称禁止修改；新增阶段目录必须走 §6 R2。
- 数据库：表名 snake_case 复数（`alert_events`），字段名 snake_case。表清单以 Stage 7 `data_model_spec.md` 为准；该文件不存在时禁止创建业务表。
- API 路径：全小写、连字符分隔、名词复数（如 `/api/v1/alert-events`）。路径清单以 Stage 7 `api_interface_spec.md` 为准。

## 2. 目录组织规范

1. 仓库顶层只允许三类业务目录：`frontend/`、`backend/`、`docs/`；其余只能是仓库级配置（`README.md`、`AGENTS.md`、`CLAUDE.md`、`.gitignore`、`.env.example`、`.pre-commit-config.yaml`、`.gitleaks.toml`、`.cursor/`、`.mcp.json`）。`docker-compose.yml` 与 `.github/` 仅 Stage 12 允许新增，不得写入业务规则；CI 门控命令必须与 §5 一致。
2. 设计资产与过程文档只存放于 `docs/`；`frontend/`、`backend/` 内禁止出现设计文档（代码目录内的 `README.md` 不算设计文档，但内容只限「如何构建 / 运行」）。
3. 可执行代码只存放于 `frontend/`、`backend/`；`docs/` 内禁止出现代码文件（`.md` 内嵌的代码示例片段不算）。
4. 阶段产物只进对应 `docs/NN_*` 目录；跨阶段引用使用相对路径链接。
5. `backend/`、`frontend/` 的内部结构按 Stage 6 已定稿架构落地；禁止预建设计未规定的模块目录（各目录占位 `README.md` 除外）。
6. Stage 11 之前：`frontend/` 与 `backend/` 除 `README.md` 外禁止新增实现文件。

**判定标准**：出现上述约定之外的顶层条目、或文件类型与所在目录不符，即不合规，当次提交必须移正后重新提交。

## 3. Git 分支策略与 Commit 规范

### 3.1 分支策略

- `main`：唯一长期分支，必须始终保持可合入状态。Stage 11 起必须保持「可构建」；有部署方案后必须保持「可部署」。
- 代码变更：从 `main` 切出 `feat/<slug>` 或 `fix/<slug>`（slug 全小写连字符，可带阶段号，如 `feat/stage11-alert-api`），完成后 squash merge 回 `main` 并删除分支。
- 文档与规范变更：可直接提交 `main`。
- 禁止长期存活的开发分支：分支创建后 5 个工作日内未合并的，必须在 `change_log.md` 说明原因，否则删除分支。

### 3.2 Commit 规范（Conventional Commits）

格式：`<type>(<scope>): <subject>`

- **type 限定**：`feat` `fix` `docs` `style` `refactor` `test` `chore` `perf` `ci` `build` `revert`
- **scope 限定**（可省略）：`frontend` `backend` `docs` `setup` `deploy`
- **subject**：中文祈使句，不超过 72 字符，结尾不加句号。
- 一个 commit 只做一件事；涉及行为变更或非平凡修改必须写 body 说明动机与影响。
- **判定标准**：commit subject 匹配正则 `^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\((frontend|backend|docs|setup|deploy)\))?: .{1,72}$`，不匹配即不合规。

### 3.3 提交门禁

- pre-commit（含 gitleaks 秘密扫描）必须通过；禁止 `git commit --no-verify`。
- 代码类 commit 合入前必须通过 §5 的验证命令。
- Stage 11 之前的文档/规范 commit：`git status` 不得出现 `frontend/` 或 `backend/` 下除 `README.md` 以外的实现文件。

## 4. AI 生成代码 Review Checklist

> 执行版本对应流程手册附录 D（忠实性 / 安全性 / 正确性 / 质量 / AI 幻觉信号）。手册未入库；以本节为唯一执行清单。手册入库后如有出入，走 §6 R2 同步。

AI 生成（或 AI 辅助）的代码合入前逐项核对；**任何一项为「否」⇒ 不得合入**。设计文档尚未产出的条目标 N/A，并在 `code_review_report.md` 写明原因；N/A 不得用来跳过安全性与幻觉检查。

### 4.1 忠实性（附录 D.1）

- [ ] **范围一致**：实现范围与当次 Task Brief 完全一致，无擅自扩展。
- [ ] **契约一致**：API 字段与 `docs/07_backend_design/api_interface_spec.md` 逐字段一致；该文件不存在时本项为 N/A，且禁止新增业务 API。
- [ ] **规则一致**：业务规则与 `docs/03_problem_modeling/business_model.md` 中的 BR 编号一致，无「理解性改写」；该文件不存在时本项为 N/A。
- [ ] **无幽灵产物**：无规范外的端点、实体、组件、配置。

### 4.2 安全性（附录 D.2）

- [ ] **无硬编码密钥**：无 Token / 连接串 / `.env` 值。
- [ ] **输入校验**：外部输入全部校验（注入 / XSS）。
- [ ] **鉴权不在前端单方面判定**：权限以后端为准；前端置灰不是安全边界。
- [ ] **敏感字段**：不出现在响应与日志。
- [ ] **无未审计新依赖**：本次变更未引入未经批准与审计的第三方依赖（红线 R4）。

### 4.3 正确性（附录 D.3）

- [ ] **边界条件**：空值 / 极限 / 并发 / 重复提交有显式处理。
- [ ] **多步写入**：有事务或补偿。
- [ ] **异常不静默吞掉**：错误信息不泄露内部栈、SQL、密钥、Prompt 原文。
- [ ] **无无跟踪的 TODO**：残留 TODO 必须带编号并写入阶段文档。

### 4.4 质量（附录 D.4）

- [ ] **测试覆盖本次变更且通过**：命令见 §5。
- [ ] **命名与结构符合本文 §1–§2**，无死代码。
- [ ] **视觉值来自 Design Tokens**：`docs/09_figma_highfi/design_tokens.json` 不存在时禁止自造色值/字号体系；本项为 N/A 仅当本次不含 UI。
- [ ] **Commit 粒度与 §3 一致**。

### 4.5 AI 幻觉信号（附录 D.5）

- [ ] **引用的依赖 / API / 配置全部真实存在**（对照 `uv.lock` / `package-lock.json` / `.env.example`）。
- [ ] **注释与链接真实有效**。
- [ ] **测试断言的是真实行为**，不是为过测试而写。
- [ ] **无被注释掉的失败尝试代码残留**。

## 5. 测试与验证要求

> 以下命令中的 uv 项目与 npm scripts 于 Stage 11 代码初始化时提供。在此之前本节用于验证「未引入实现代码」。

### 5.1 后端（pytest）

- 框架：pytest + pytest-asyncio + httpx；测试位于 `backend/tests/`（Stage 11 创建）。
- **必须**：
  - service / domain 层每个新增函数至少 1 条正例测试；含分支或外部调用（IO / 网络 / DB）的函数额外至少 1 条异常路径测试。
  - 每个 API 端点至少 1 条 happy-path 集成测试（httpx AsyncClient）。
  - bug 修复先写复现测试（先失败、后通过），测试与修复在同一 commit。
- **合入前验证命令（必须全部通过）**：在 `backend/` 执行 `uv run ruff check . && uv run ruff format --check . && uv run mypy app && uv run pytest`

### 5.2 前端（Vitest + React Testing Library）

- **必须**：组件交互逻辑（事件、状态流转）有测试；纯函数全量测试。
- **合入前验证命令（必须全部通过）**：在 `frontend/` 执行 `npm run lint && npx tsc --noEmit && npm run build && npm run test`

### 5.3 Stage 11 之前

- **必须**确认本次变更未新增实现代码：`git status` / `git diff --name-only` 中 `frontend/`、`backend/` 仅允许 `README.md`。
- **必须** `pre-commit run --all-files`（含 gitleaks）零告警。

### 5.4 通用

- 无法本地验证的变更（如部署配置），必须在对应阶段输出文档中记录验证方式与验证人。
- **判定标准**：验证命令输出零失败；CI（Stage 12 引入）为绿。

## 6. AI 使用红线

违反任一红线的处置：立即停止当前 AI 操作，本次产生的变更全部回滚；已合入 `main` 的，revert 并在 `docs/13_changes/change_log.md` 登记。

### R1 禁止将密钥、生产数据、用户隐私数据提供给 AI

- 密钥只存在于本地 `.env`；任何 AI 会话、提示词、粘贴内容、日志、示例中不得出现 `.env` 的值。
- 需要 AI 协助处理数据问题时，必须使用脱敏或合成样例（不含真实用户标识、真实告警正文中的主机名/账号/令牌）。
- **执行判定**：提交内容经 gitleaks 扫描零命中；对话与文档抽查无密钥值、无真实 PII。命中任一即违规。

### R2 禁止 AI 静默修改已冻结的设计资产

- 已冻结资产判定见根 `AGENTS.md` §5。
- 修改冻结资产的前置条件：① 先在 `docs/13_changes/change_log.md` 登记；② 获用户显式批准；③ 修改后在回复中展示变更 diff。三者缺一即视为静默修改。
- **执行判定**：冻结文档的任意 git 变更必须能对应到 change_log 记录；对应不到的必须 revert。

### R3 禁止提交未运行验证的 AI 生成代码

- 任何 AI 生成代码在提交前必须实际运行 §5 验证命令且零失败。
- 无法运行验证的代码不得合入 `main`（只能留在分支并在阶段输出文档中说明阻塞原因）。
- **执行判定**：commit message body 或阶段输出文档中记录验证命令与结果；抽查必须能复现验证输出。缺少记录即违规。

### R4 禁止 AI 自行引入未审计的第三方依赖

- 新增运行时依赖的固定顺序：获用户批准 → 审计（维护活跃度、License 兼容、无未修复的安全通告）→ `uv add` / `npm install` 落锁。
- 依赖文件（`pyproject.toml`、`uv.lock`、`package.json`、`package-lock.json`）的变更必须独立成 commit。
- **执行判定**：依赖文件变更的 commit 无法对应到事先批准记录（会话中的用户明确同意，或 `change_log.md` 条目）的，视为违规。
